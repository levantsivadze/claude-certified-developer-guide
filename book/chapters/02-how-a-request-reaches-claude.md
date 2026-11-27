# Chapter 2: Four Parts to Every Call, and Three Places It Breaks

**Summary**: *Every request to Claude has the same four parts, and every failure worth debugging happens in one of three places between them. Keep those places separate and many scenarios answer themselves: which failures are worth retrying, which must be fixed before the call goes out, and which arrive with a perfectly healthy status code. Mix them up and you'll waste an afternoon adding retries to a broken request, or filing a network ticket about an answer the model gave on purpose.*

## Every Call Has Four Parts, and the Fourth Needs Interpreting

Write a call out by hand once, just to see it. Not because you'll ship it that way, but because everything a client library does is one of these **four parts**. When something fails inside a part nobody has looked at, it gets blamed on whichever part people *have* looked at.

Claude is a **REST API** at `https://api.anthropic.com`. A message goes to **`POST /v1/messages`**.<sup>[1]</sup> That's the **address**. Then come **three required headers**:<sup>[1]</sup><sup>[2]</sup>

- **A credential:** `Authorization: Bearer <token>`, where the token is your API key or a short-lived access token. The older `x-api-key` header still works as a fallback.
- **An API version:** `anthropic-version`, currently `2026-10-21`.
- **A content type:** `content-type: application/json`.

Then a JSON **body**. Then a **response**, which is where developers stop paying attention and where the interesting failures happen.

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Authorization: Bearer $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2026-10-21" \
  -H "content-type: application/json" \
  -d '{"model": "claude-opus-5-5", "max_tokens": 1024,
       "messages": [{"role": "user", "content": "Hello, Claude"}]}'
```

| Part | What it carries | What its failure looks like |
|---|---|---|
| Address | Endpoint and HTTP method | A `404` on a path that looks right |
| Headers | Credential, API version, content type | A `401` or `403`, the same on every attempt |
| Body | The JSON request | A `400`, or a `413` if it's too large |
| Response | Status code, headers, JSON body | Nothing at all, until something downstream breaks |

The last row matters most. A **status code** tells you whether the exchange completed. The **body** tells you what came back. Those are **two separate questions**, and code that checks the first and assumes the second is the most common bug on this material.

## The Client Library Is Someone Else's Maintained Copy of That Boilerplate

Anthropic publishes official SDKs for **seven languages**: Python, TypeScript, C#, Go, Java, PHP and Ruby. There's also an **`ant` command-line tool** for shell scripting and interactive use.<sup>[1]</sup><sup>[3]</sup> An SDK sends the required headers for you, including the API version that matches its major version.<sup>[2]</sup>

The SDK and a hand-written HTTP call reach the **same endpoint and the same model**. The answers aren't any better. What changes is **who maintains the parts**, and there are more of them than you'd think:

- Errors arrive as **typed exceptions**, not raw JSON. The docs say to catch those classes rather than matching on message text.<sup>[4]</sup>
- A **retry policy** is already built in (next section).
- List endpoints come with an iterator that follows the page cursor.<sup>[1]</sup>
- Non-streaming requests are checked against a **10-minute timeout** before they're sent.<sup>[4]</sup>
- The **`request-id`** of each response is exposed as a property.<sup>[4]</sup>

None of this is hard. All of it is **maintenance**, and it falls on whoever wrote it.

Three tempting alternatives, each a named mistake:

- **The hand-rolled client.** It's tempting because the first version works: a general HTTP library, three headers, a JSON body, done in twenty minutes. But you've just created a private copy of a library nobody wanted to own, and now you have to keep it in step with the API by hand, forever. When Anthropic released API version `2026-10-21`, SDK users got it by upgrading a package. Hand-rolled clients had to work through the breaking changes themselves.<sup>[5]</sup>
- **A foreign client plus a translation shim.** Anthropic offers a compatibility layer so you can test Claude through the OpenAI SDK, but it's documented for evaluation, and it differs from the native API in several ways.<sup>[3]</sup><sup>[6]</sup> A shim adds a second API's rules and one more layer to debug, in exchange for a provider switch nobody has scheduled.
- **Calling the CLI as a subprocess.** The `ant` tool is meant for shell scripting and interactive use.<sup>[3]</sup> A service that starts it once per request gives up typed errors and connection reuse, and pays for a new process on every call.

## Retries: Two by Default, and Adjustable

What happens when a call fails for a reason that has nothing to do with the request? The official SDKs answer precisely:<sup>[4]</sup>

- They **retry twice by default** (3 attempts in total). `max_retries` on the client changes this, and `0` turns retries off.
- If the response has an **`x-should-retry`** header, the SDK follows it. Otherwise it retries **connection errors, timeouts, and `408`, `409`, `429` and `500`+**.
- If there's a **`retry-after`** header, the SDK waits that long. Otherwise it uses **exponential backoff from 0.5 to 8 seconds**.
- Errors that arrive **inside a stream are not retried**.

Exam scenarios on this almost always hinge on one half of "default *and* adjustable":

| Claim about the SDK's retries | Verdict |
|---|---|
| Nothing happens unless your code does it | Wrong. A default policy is already running |
| It happens and you can't change it | Wrong. `max_retries` adjusts it, and `0` turns it off |
| Connection errors retry, rate limits don't | Wrong. `429` is retried, and it's the most common case |
| A default exists and a setting adjusts it | Correct |

The useful answer for a teammate sits between the two obvious ones: **a policy is already running whether or not anyone chose it**, and a team with a tight latency budget can turn it down or off in one line.

**A retry loop around a retry loop** is the classic mistake, and it's simple arithmetic.

> **Example.** Your code retries 3 times. Each of those calls goes through the SDK, which makes up to 3 attempts. That's up to **9 requests** to a service that has already asked you to slow down. Nobody writes this on purpose. It happens because whoever wrote the outer loop didn't know the inner one existed.

Two more traps sit close by:

- **A timeout set for the whole request doesn't restart when the SDK retries.**<sup>[7]</sup> A generous outer deadline can expire in the middle of a retry that was about to succeed.
- **Not every `429` is temporary.** A `429` caused by your usage tier's **spend cap** has **no `retry-after` header** and keeps failing until access is restored. You can recognise it by `error_code: "enforced_spend_limit_reached"` in the error body.<sup>[4]</sup> Retrying it forever turns a billing problem into an outage.

## Persistent and Two-Way Are Separate Properties, and Only One Needs a WebSocket

Teams reach for a **WebSocket** whenever someone says "real time". That's the wrong trigger. Two separate properties decide the shape: **how long the connection lasts**, and **which ends can send messages without being asked**.

| Shape | Who can send without being asked | What it costs | When it fits |
|---|---|---|---|
| Request and response | Neither end | One connection setup per exchange | A question with one answer |
| Polling | Neither end | Lots of requests, and data only as fresh as the interval | Cheap checks on something that changes slowly |
| Held-open one-way response (server-sent events) | Server only | A connection held for the length of the response | Output arriving in pieces while someone waits |
| WebSocket | Both ends | A connection to keep alive, reconnect and monitor | A long-lived exchange where both ends push |

Read down the second column and the choice becomes clear. If updates arrive **as they happen, from the other end, unasked**, and messages also go **back over the same connection**, and the relationship lasts **minutes or hours**, that's what a WebSocket is for.

**Polling dressed up as real time** is the mistake people accept without noticing. Shortening the interval buys freshness with more requests, but the interval never reaches zero, so the data is always as old as the gap between checks. The clue in a scenario is a design that improves when a **number gets smaller**.

The other substitute is subtler. An HTTP response **held open** sends data **one way**, from server to client. Claude's own streaming works this way, using **server-sent events**.<sup>[8]</sup> It's the right shape for output that arrives gradually (chapter 3). But it isn't a two-way channel: to say anything back, you have to open a new request.

Anthropic's own products show the difference between *who opens* a connection and *who sends over it*. An **MCP tunnel** connects Claude to servers inside a private network using an **outbound-only connection**, so no inbound firewall port needs to be opened.<sup>[9]</sup> The traffic then runs inside the tunnel's WebSocket stream.<sup>[10]</sup> Confusing these two facts is how a security review ends up demanding a firewall change nobody needs.

## Three Places a Call Breaks, and Only the Middle One Is Worth Retrying

Every failure lands in one of **three places**. **Naming the place first** is the whole diagnostic skill, because each needs a different fix, and each gets worse under the fix meant for a neighbour.

| Where it breaks | How it shows up | What fixes it | What makes it worse |
|---|---|---|---|
| **Integration** (your request) | `400`, `401`, `403`, `404`, `413`. The same on every attempt | Fix the request, credential or size | Retrying, which pays again for the same answer |
| **Transport** (getting it there) | Connection errors, `429`, `500`, `529`. On and off, worse under load | Limited retries with backoff, honouring `retry-after` | A tighter loop with no limit on attempts |
| **Output** (what came back) | HTTP `200`, but the content is unusable | Validation, a schema, a fallback model | Checking the status code and stopping there |

**Integration** failures belong **entirely to you**. Their signature is that they're **the same every time**: the same bytes fail the same way at 3 a.m. as they did in testing. A `413` for a request over the size limit (128 MB on the Messages API) will never succeed on the fourth try.<sup>[4]</sup>

**Transport** failures are what the retry policy exists for. A `529` means the service is temporarily overloaded. A `500` is worth retrying unless the response says `x-should-retry: false`.<sup>[4]</sup>

**Output** failures hurt the most, because **everything before them worked**. A **refusal arrives as a normal `200`** with **`stop_reason: "refusal"`**. The usual fix is to send the same request to a different model, not to treat it as an error.<sup>[11]</sup> Streaming makes it trickier: an error can arrive **after the API has already returned `200`**, so normal HTTP error handling never sees it.<sup>[4]</sup>

> **Example.** A nightly job summarises support tickets and writes them to a database. It reports no errors for a week. Then someone notices that 3% of rows say "I can't help with that request." Every one of those calls returned `200`. The code checked the status, saw success, and saved the text. The fix is to check `stop_reason` before saving, and to route refusals to a fallback model or a review queue.

One free habit closes this out. Every response has a **`request-id`** header, and error bodies carry the same value. Support asks for it because it's the only thing that identifies a single call.<sup>[4]</sup> **Log it on every failure.**

## The Error Codes, and Where Each One Belongs

The **place a call broke** is what to bring into a question. The codes are just a lookup. But a half-remembered lookup is worse than none, because it confidently sorts a code into the wrong place and then applies the wrong fix.<sup>[4]</sup>

**These say your request, credential or account is wrong.** Another attempt won't help, with one exception (`409`):

| Code | Error type | What it means |
|:----|:------------------|:--------------------------|
| `400` | `invalid_request_error` | Something's wrong with the request's format or content. Also the catch-all for unlisted 4xx codes, and returned when an org or workspace spend limit is reached |
| `401` | `authentication_error` | The key is malformed, revoked or expired |
| `402` | `billing_error` | A billing or payment problem |
| `403` | `permission_error` | The key is valid but lacks permission for this resource |
| `404` | `not_found_error` | Wrong endpoint path or resource ID |
| `409` | `conflict_error` | A conflict with the resource's current state, such as a concurrent edit or a duplicate unique value. Resolve it, then retry |
| `413` | `request_too_large` | The request is over the size limit |

**These say the service, or your pace, is the problem.** Waiting is part of every fix:

| Code | Error type | What it means |
|:----|:------------------|:--------------------------|
| `429` | `rate_limit_error` | A rate limit, monthly spend cap, or Claude Code workspace spend limit |
| `500` | `api_error` | An unexpected error inside Anthropic's systems |
| `504` | `timeout_error` | The request timed out while being processed |
| `529` | `overloaded_error` | The API is temporarily overloaded |

Three of these deserve a second look:

- **`400` is a catch-all, not a diagnosis.** A spend limit you set on your own organisation or workspace returns a `400` (a Claude Code workspace limit can return `429` instead). So a `400` doesn't prove the body is malformed.
- **`401`, `402` and `403` all mean "you can't proceed", for three unrelated reasons**: a bad key, an unpaid account, or a good key without this permission. Three different fixes, and none of them is retrying.
- **`409` is the odd one out.** It looks like an integration error, but the SDK retries it by default. That makes sense once you see what a conflict is: a state that another writer may already have released.

## How the Exam Builds Wrong Answers

Wrong answers on integration rarely describe something impossible. They describe a real technique, correctly, but **applied to the wrong place**. Four assumptions produce most of them:

- **The SDK either does nothing or can't be changed.** Options split into "your code must do this itself" and "the SDK does this and you can't change it". Both miss what an SDK actually offers: **a documented default with a setting to adjust it**.
- **Freshness bought with request volume, sold as a live connection.** Polling is fine for slow-changing data. It's the wrong shape when the other end knows first, because latency stays tied to whatever interval someone picked.
- **A status code read as a verdict on the content.** Options that treat `200` as success and everything else as failure ignore a whole category of failure. Watch for scenarios where something broke downstream of a call that never errored.
- **A heavier integration preferred over the maintained one.** Building on a general HTTP client, wrapping another provider's library, or shelling out to a CLI all look like engineering judgement. Ask what each one now has to **keep up to date by hand** as the API changes.

Match on what the scenario is doing, not the industry it's set in:

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| A service pushes updates continuously and the app also sends messages back | A protocol built for a persistent two-way connection | Any option whose freshness depends on how often you ask |
| A teammate asks whether the app needs its own retry logic | A default policy exists and the client exposes it as a setting | Saying there's none, or that it can't be changed |
| There's an official SDK for the language and the team is starting an integration | Use the SDK and its documented pattern | Rebuilding headers, retries and parsing on a general HTTP library |
| A call fails the same way on every attempt | The request, credential or size is wrong, and it's yours to fix | Adding backoff to a failure that will never change |
| Failures come and go and get worse under load | Limited retries with backoff, honouring `retry-after` | A tighter loop, or retrying a spend cap forever |
| Nothing errored, but a later step broke | Check the response body and `stop_reason`, then validate | Treating a healthy status code as a healthy answer |
| An option sorts a status code by its number alone | Ask whether another attempt could change the outcome, then fix the right place | Treating `400` as proof of a bad body, or `409` as unretryable |
| Portability across model providers is the stated reason | Weigh the extra layer against a switch nobody has scheduled | A foreign client plus a translation shim |

The habit to keep: **name where it broke before naming the fix.** Say out loud whether the failure is in what you **sent**, in **getting it there**, or in what **came back**, and about half the options stop looking plausible.

## What to Remember

- Every call has **four parts**: **address, headers, body, response**. The headers are a **credential** (`Authorization: Bearer`, or the older `x-api-key`), **`anthropic-version`** (currently `2026-10-21`) and **`content-type`**. The response has a status code, headers and a body, and you have to check the status and the body separately.
- **Official SDKs exist for seven languages** and send the required headers for you. They reach the **same endpoint and model** as a hand-written call. What differs is **who maintains the parts** as the API changes.
- **SDKs retry twice by default** (3 attempts), follow **`x-should-retry`** and **`retry-after`**, and otherwise back off exponentially. **`max_retries`** adjusts it, and `0` turns it off. **Errors inside a stream aren't retried.**
- **Retried by default**: connection errors, timeouts, **`408`, `409`, `429`, `500`+**. A **retry loop around the SDK's** multiplies attempts, and an **outer deadline doesn't restart** when the SDK retries.
- A **spend-cap `429`** has no `retry-after` and keeps failing until access resumes. Retrying it forever turns a **billing problem into an outage**.
- **The codes**: `400` invalid request, `401` authentication, `402` billing, `403` permission, `404` not found, `409` conflict, `413` too large, `429` rate limit, `500` internal, `504` timeout, `529` overloaded. **`409`** is the only request-side code the SDK retries, because the conflict can clear.
- **Persistent and two-way are separate properties.** Polling keeps data as stale as its interval. A held-open response (server-sent events) only pushes **server to client**. A **WebSocket** lets both ends send unprompted on one connection.
- **Three places a call breaks**: **integration** (4xx, same every time, yours to fix), **transport** (connection errors and 5xx, intermittent, retry with backoff), and **output** (**`200` with unusable content**, fixed by validation or a different model).
- A **refusal is a normal `200`** with **`stop_reason: "refusal"`**, and a **streaming error can arrive after the `200`**. Error handling that only checks status codes misses a whole category.
- Every response carries a **`request-id`**. **Log it on failure.** It's the only identifier for a single call.

## Endnotes for this chapter

1. The Claude API is a RESTful API at `https://api.anthropic.com`; the `Authorization` header takes a bearer token that is an API key or a short-lived access token, with `x-api-key` kept as a legacy fallback; official SDKs exist for Python, TypeScript, C#, Go, Java, PHP and Ruby; and the SDKs provide auto-paginating iterators. Anthropic, *API overview* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/api/overview](https://platform.claude.com/docs/en/api/overview)

2. Every request must send an `anthropic-version` header, for example `2026-10-21`, and the client SDKs set it for you. Anthropic, *API versions* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/api/versioning](https://platform.claude.com/docs/en/api/versioning)

3. Anthropic publishes an `ant` command-line tool for shell scripting and interactive use, client SDKs in seven languages, and framework-specific libraries that expose Claude through another framework's API surface and are not general-purpose Messages API clients. Anthropic, *CLI, SDKs, and libraries* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/cli-sdks-libraries/overview](https://platform.claude.com/docs/en/cli-sdks-libraries/overview)

4. HTTP error codes and types; request size limits of 128 MB for the Messages API; SDK retries defaulting to 2 (3 attempts), configured with `max_retries`, following `x-should-retry` when present and otherwise retrying connection errors, timeouts and 408, 409, 429 and 500+; waits following `retry-after` or exponential backoff from 0.5 to 8 seconds; spend-cap 429s with no `retry-after` and the `enforced_spend_limit_reached` error code; errors after a 200 over streaming, which the SDK doesn't retry; the 10-minute validation for non-streaming SDK requests; and the `request-id` header and `request_id` field. Anthropic, *Claude API errors* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/api/errors](https://platform.claude.com/docs/en/api/errors)

5. API version 2026-10-21 and its breaking changes; SDK users move to it by upgrading to the first major version that sends it, while callers without an SDK set the header and update their code themselves. Anthropic, *Migrating to API version 2026-10-21* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/api/migration-2026-10-21](https://platform.claude.com/docs/en/api/migration-2026-10-21)

6. Anthropic provides a compatibility layer that lets you use the OpenAI SDK to test the Claude API, with documented limitations and differences from the native API. Anthropic, *OpenAI SDK compatibility* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/cli-sdks-libraries/libraries/openai-sdk](https://platform.claude.com/docs/en/cli-sdks-libraries/libraries/openai-sdk)

7. A retried request does not restart a context timeout set for the request lifecycle. Anthropic, *Go SDK* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/cli-sdks-libraries/sdks/go](https://platform.claude.com/docs/en/cli-sdks-libraries/sdks/go)

8. Setting `"stream": true` on a message returns the response incrementally using server-sent events. Anthropic, *Streaming messages* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/streaming](https://platform.claude.com/docs/en/build-with-claude/streaming)

9. MCP tunnels connect Claude to MCP servers inside a private network over an outbound-only connection, with no inbound firewall ports opened. Anthropic, *MCP tunnels* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview)

10. The inner TLS handshake between Anthropic's backend and the customer's proxy is carried inside the tunnel's WebSocket stream, and "outbound-only" describes the connection rather than the requests carried over it. Anthropic, *Architecture and components* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/concepts](https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/concepts)

11. A refusal is a successful HTTP `200` response with `stop_reason: "refusal"` rather than an error, and the same request can usually be sent to another Claude model. Anthropic, *Refusals and fallback* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)
