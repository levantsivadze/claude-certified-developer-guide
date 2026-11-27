# Chapter 3: Messages, Streaming, Batches: Who Is Waiting Decides

**Summary**: *Three requests can use the same model, prompt and document and still belong on three different endpoints. What separates them is who's waiting for the answer. Get that wrong and you either pay full price for work nobody was watching, or leave someone staring at an empty screen while a perfectly good answer is being built out of sight. This chapter covers the API organised around that question, from one message to forty thousand.*

## Decide Who Is Waiting Before Anything Else

Three requests, all with the same model, instructions and attached document:

1. One was typed by someone **watching a cursor blink**.
2. One came from a service that **must return an HTTP response within two seconds**.
3. One is number 8,431 in an **overnight run** of forty thousand, and the only deadline is a report due at 8 a.m.

Nothing in the **prompt** tells them apart. Everything about the **delivery** does.

This book calls this **the waiting party**: whoever's clock the request is running on. It might be a person watching a screen, a caller blocked on a return value, or nobody at all until a deadline hours away. **Decide it once, early**, and four decisions follow together. Decide it late and you'll be retrofitting, because the endpoint, error handling, cost model and interface all depend on it.

| Who's waiting | Shape | What you get | What it costs |
|---|---|---|---|
| A person watching output appear | A **streamed** message | Visible progress within a second | Event handling, and pieces to reassemble |
| A caller blocked on the result | One ordinary request | One complete object to parse | The caller waits for the whole generation |
| A caller doing other work meanwhile | Many ordinary requests, sent concurrently | Throughput without a queue | Your rate limits, hit at your own pace |
| Nobody, until a deadline hours away | A **batch** | Half the token price, higher throughput | Results in hours, not seconds |

Compare the last two rows carefully, because scenarios pull them apart. Both are called **asynchronous**, but they solve different problems. **Concurrent calls keep your process busy** while requests are in flight, but each response still arrives at normal speed. **A batch gives up that speed** in exchange for a discount.

## One Request Is a Model, a Limit and a List of Turns

The request body is smaller than people expect: a **model ID**, a **`max_tokens`** limit on the response, and **`messages`**, an ordered list of turns, each with a `role` and `content`. **Standing instructions** go in a separate top-level **`system`** field, not inside a turn.<sup>[1]</sup>

**Content is a list of typed blocks**, not a plain string. The block type is how the API grows: it's why one endpoint handles text, images, documents, tool calls and reasoning. A response comes back the same way: an `id`, `role: "assistant"`, a `content` array of blocks, a **`stop_reason`**, and a **`usage`** object counting tokens in and out.<sup>[1]</sup>

```json
POST /v1/messages
{
  "model": "claude-opus-5-5",
  "max_tokens": 16000,
  "system": "You are a support assistant for Acme Bank.",
  "messages": [
    { "role": "user", "content": [ { "type": "text", "text": "Why was I charged twice?" } ] }
  ]
}
```

The property that shapes everything else: **the Messages API is stateless**. You send the **whole conversation history on every call**.<sup>[1]</sup> A multi-turn conversation is a list you keep and grow: add the assistant turn you got back, add the next user turn, and send the whole thing again.

Two consequences follow, and both are tested:

- **Earlier turns don't have to come from Claude.** You can write **synthetic assistant turns**, for example to show the model an example exchange.<sup>[1]</sup> One older trick no longer works: **prefilling** the start of Claude's answer in a final assistant turn returns a `400` error on current models.<sup>[2]</sup> Use structured outputs or instructions instead (chapters 5 and 6).
- **The history is the bill.** Every turn you carry is billed as input tokens again on the next call, so a conversation gets more expensive per exchange as it grows, even when the questions stay short. Chapter 7 covers managing a filling window, and chapter 8 covers the arithmetic.

> **Example.** Each turn in a support chat adds about 500 tokens. On turn 1 you send 500 input tokens. On turn 10 you send about 5,000, because turns 1 to 9 ride along. Over the whole 10-turn chat you've paid for about 27,500 input tokens, not 5,000. Prompt caching (chapter 8) makes the repeated part much cheaper, but it's still sent.

Two newer features soften this, and both are worth knowing:

- **Mid-conversation system messages.** You can add a message with `"role": "system"` partway through the conversation, right after a user turn. It adds an instruction without changing the top-level `system` field, so the cached prefix stays intact.<sup>[3]</sup>
- **Message threads (beta).** The server can hold the conversation for a short time, so later requests send only the new messages and the ID of the previous response. This mainly improves time to first token in long agent loops. A thread expires 5 minutes after its last request and is deleted within an hour of its last use.<sup>[4]</sup> It's a speed feature, not long-term storage: your app still owns the conversation.

## A Tool Result Is Matched by ID, Not by Position or Description

The strictest part of the API is the **tool round trip**. Two systems have to agree on which answer belongs to which question, and only one of them reads English.

1. Claude returns **`stop_reason: "tool_use"`** and one or more **`tool_use`** blocks. Each has an `id`, a `name`, and an `input` object matching the tool's schema.
2. You run the tool.
3. You send a **new user turn** containing **`tool_result`** blocks. Each one repeats the **`tool_use_id`** of the call it answers, puts the result in `content`, and sets `is_error` if the tool failed.<sup>[5]</sup>

```json
{ "role": "user", "content": [
  { "type": "tool_result", "tool_use_id": "toolu_01A9", "content": "Balance: $1,240.00" },
  { "type": "tool_result", "tool_use_id": "toolu_01B2", "content": "No open disputes" }
] }
```

Three rules come with this, and each one matters:

- **That user message contains only the result blocks.** Adding your own text after them can produce an empty response.<sup>[6]</sup> Send extra input in a separate user message after the turn ends.<sup>[5]</sup>
- **Keep the `tools` array the same** when you continue.<sup>[6]</sup>
- **Send the assistant content back exactly as it arrived**, including any thinking blocks. Keep the conversation append-only.<sup>[7]</sup>

Two named mistakes live here, and both look reasonable in a code review:

- **The result written as prose.** You take the tool output, write it as an English sentence, and send it as an ordinary user message. A person can read it fine, but you've **lost the ID**, so nothing connects the answer to the call. With two tools running at once, nothing says which is which.
- **The result put in the assistant turn.** The reasoning goes: the model asked for it, so it belongs on the model's side. But the assistant turn is the record of what *Claude* produced. Putting outside data there claims Claude produced it, which is false, and it leaves the original `tool_use` block unanswered.

Chapter 9 covers how tools are **described and chosen**. This section is just the wiring.

## `stop_reason` Tells You What to Do Next

Every successful response has a **`stop_reason`**. It's what tells you whether you have a finished message or a fragment.<sup>[6]</sup> Group the values by **what they require you to do**, because that's what scenarios actually ask about:

| What you need to do | Values | How |
|---|---|---|
| Nothing. The turn is complete | `end_turn`, `stop_sequence` | Use the content. `stop_sequence` tells you which sequence fired |
| Send a continuation | `tool_use`, `pause_turn` | For `tool_use`, send `tool_result` blocks. For `pause_turn`, send the response back unchanged |
| Treat the answer as cut short | `max_tokens`, `model_context_window_exceeded` | Mark the output incomplete, or raise the limit and continue |
| Send the request somewhere else | `refusal` | Read `stop_details` and retry on a fallback model |

That's **four kinds of action for seven values**. Two deserve a closer look.

**`pause_turn` and `tool_use`** both mean the turn isn't finished, but you continue them differently, and that's the trap. `pause_turn` comes back when a **server-side tool loop** (such as web search) hits its iteration limit, 10 by default. You continue by appending the response's content unchanged as an assistant message and resending. `tool_use` means **Claude is waiting for you**, and you continue with your results.<sup>[6]</sup>

**`refusal`** never looks like trouble. It arrives as an **ordinary HTTP `200`**. `stop_details` names the policy category that triggered it. You can often get an answer by retrying on another model. Some models also support a server-side fallback, but Claude Haiku 5.5 doesn't, so you need your own fallback there. One exception: a `reasoning_extraction` refusal should be fixed by changing the prompt, not by switching models.<sup>[6]</sup>

**Reading the status code as the verdict** is still the main mistake, and `stop_reason` is the fix. A pipeline that checks for `200` and parses the content will pass on a truncated document, an unanswered tool call and a refusal. In a log, all three look like ordinary short answers.

## Streaming Changes When the Words Appear, Not What They Cost

Set **`"stream": true`** and the response arrives piece by piece over **server-sent events**.<sup>[8]</sup> Each event has a type and a JSON payload, in a predictable order:

```
message_start          ← empty message shell (stop_reason is null here)
  content_block_start
  content_block_delta  ← repeated, one piece at a time
  content_block_stop
  ... (more blocks)
message_delta          ← stop_reason and usage arrive here
message_stop
(ping events can appear anywhere)
```

Three facts matter more than the event names:

- **`stop_reason` is `null` in `message_start`** and only arrives in **`message_delta`**.<sup>[6]</sup> Code that reads the first event and moves on has read the field before it was filled in.
- **Errors can arrive inside the stream.** Under heavy load, an `overloaded_error` can appear as an `error` event on a connection that **already returned HTTP `200`**. A non-streamed request would have failed with a `529` instead.<sup>[8]</sup> The SDK doesn't retry errors inside a stream.<sup>[9]</sup> A consumer that only handles text pieces will treat it as the end of a short answer.
- **New event types may be added**, so skip unknown types rather than failing.<sup>[8]</sup>

Streaming also has a use that has nothing to do with anyone watching. The **SDKs can stream internally** and hand back one complete message, like an ordinary call. They require this for requests with a large `max_tokens`, because a long generation on one non-streamed call runs into HTTP timeouts.<sup>[8]</sup>

**Using streaming to speed up a backend job** is the named mistake. The first words appear sooner, but the **total generation time doesn't shrink**, and the token cost is the same. A backend job that waits for the whole message before acting has gained event-handling code and nothing else. Streaming answers one requirement: **a person is watching**.

## Images and Documents Are Blocks Too, With Three Ways to Send the Bytes

There's no separate vision endpoint. An **`image`** content block sits in a user turn next to the text. You can supply the bytes three ways: inline **`base64`**, a **`url`**, or a **`file_id`** from the Files API. JPEG, PNG, GIF and WebP are accepted, up to **8000 × 8000 pixels**.<sup>[10]</sup>

Several images in one request are **analysed together**, which is what makes page-by-page document work possible. **Label each image** in the text so you can refer to it later. Images from earlier turns stay available, so a follow-up question doesn't need them resent.<sup>[10]</sup>

One limit is easy to hit and hard to diagnose. With **more than 20 image blocks** in a request, a **stricter size limit applies to every image** in it, including resent images and images inside tool results. Oversized images are **rejected**, not scaled down.<sup>[10]</sup>

**PDFs go in as `document` blocks**, using the same three routes. The cost model is the part to remember: each page is turned into an image *and* read as text, so a **PDF is billed both ways**. **Page count drives the cost**, not file size.<sup>[11]</sup>

That's where the **Files API** helps. **Upload once**, get a **`file_id`**, and refer to it in as many later requests as you like instead of re-encoding the file each time.<sup>[12]</sup>

> **Example.** A user uploads a 40-page scanned contract and asks ten questions about it. If you send it as base64 on every turn, you upload the same multi-megabyte file ten times. With the Files API you upload it once and each turn sends a short `file_id`. (It's still billed as input tokens on each turn; caching helps there.)

Check **availability** before designing around it: the Files API is offered on the first-party API, and only in beta on some cloud platforms.<sup>[13]</sup>

## Batches Cost Half as Much, but You Match the Results Yourself

The **Message Batches API** is the last row of the opening table. You submit requests together, they're processed asynchronously, and they're **billed at 50% of normal prices**. Most batches finish **within an hour**, and any batch **not finished in 24 hours expires**. A batch can hold up to **100,000 requests or 256 MB**.<sup>[14]</sup>

The pattern has **four steps**, and the last two are yours:

```
create   POST a list of requests; each has a custom_id and a normal Messages request
poll     processing_status goes from in_progress to ended
collect  download the results: a JSONL file, one result per line
match    join each result back to your data by its custom_id
```

**Results come back in any order**, so match them by **`custom_id`, never by position**.<sup>[14]</sup> You choose the ID (1 to 64 letters, digits, hyphens or underscores). Using an ID that already means something in your system, like an order number, is the difference between a join and a guess.

Each result has one of **four types**, and it matters for billing as much as for control flow:

| Result type | Meaning | Billed? |
|---|---|---|
| `succeeded` | The message was created | Yes |
| `errored` | Invalid request or server error | No |
| `canceled` | The batch was cancelled before this request ran | No |
| `expired` | The 24-hour limit was reached first | No |

One failed request doesn't affect the others. Results can be downloaded for **29 days** after the batch was created. You can **cancel** a batch midway, and it may still return partial results.<sup>[14]</sup>

Some parameters are **rejected in a batch**, and each rejection makes sense: **`stream`** (results come back as a file), **fast mode** (it tunes real-time latency, which doesn't exist here), and server-side **fallbacks**.<sup>[14]</sup>

**Prompt caching** still works and **stacks with the batch discount**, but because batch requests run in parallel, cache hits are **best-effort, typically 30% to 98%**. Anthropic recommends the 1-hour cache for shared context, because batches can run longer than 5 minutes.<sup>[14]</sup> Treat the caching saving as a bonus, not a guarantee.

Two mistakes, mirror images of each other:

- **Choosing a batch for speed.** "Asynchronous" doesn't mean "faster". A batch never returns any single answer sooner. If a caller is waiting on one result, batching makes the wait worse.
- **Running a synchronous loop at batch volume.** This is more common: it's how most overnight jobs start. Tens of thousands of sequential calls hit your rate limits, tie up the process, and cost **twice** what the same work costs as a batch. The requirement that points to a batch is **high volume** plus **no clock on any single answer**, and **both** must be true.

## Cloud Providers Keep the Message Shape and Change What's Around It

Many companies use Claude through **Amazon Bedrock**, **Google Cloud**, **Microsoft Foundry** or **Claude Platform on AWS**. The reasons are rarely technical: an existing contract, a security boundary compliance has already approved, or a wish to keep one billing relationship. The requirement comes from the organisation, and it **applies from the first call**.

**What stays the same** is the part you spend most time on: content blocks, `stop_reason`, tool use and streaming. Anthropic's own SDKs support these platforms directly.<sup>[15]</sup> Bedrock uses the same Messages API shape, running on AWS-managed infrastructure inside the AWS security boundary.<sup>[16]</sup>

**What changes** is everything around the message:

- **Authentication** is the platform's, not an Anthropic API key.
- **Endpoints and model IDs** are the platform's.
- **Billing** goes through the platform's marketplace, which is often why the requirement exists.<sup>[17]</sup>
- On **Google Cloud**, the **model goes in the URL** instead of the body, and **`anthropic_version` goes in the body**.<sup>[15]</sup>
- **Features vary.** Something on the first-party API may be in beta or missing on a given platform.<sup>[13]</sup> For example, fast mode for Opus 5 was available on the Claude API only.<sup>[18]</sup>

There's one good engineering answer and two tempting wrong ones:

- **Right:** use the **required platform from the first commit**, with a **thin layer** between your app and the provider so the differences live in one place.
- **Wrong: migrate later.** Build against the direct API now and move later. Every request until then breaks the agreement, and the rework is guaranteed.
- **Wrong: build both.** Two code paths maintained forever, and production still runs the one the contract forbids.

## How the Exam Builds Wrong Answers

Wrong answers on API mechanics are almost never made up. They name a real endpoint, parameter or technique, and **aim it at a requirement it doesn't serve**. Four assumptions produce most of them:

- **"Asynchronous" read as "fast".** The option treats a batch as a performance improvement. A batch has high throughput, but **each individual answer arrives later**. It's only correct when the volume is large and no one is waiting on individual results.
- **The wrong kind of fix for a delivery problem.** A requirement about *when output reaches someone* gets answered with caching, a bigger token limit, a schema or a bigger model. Each is real, but none makes a response arrive in pieces, which is all the requirement asked for.
- **A structural link replaced by a description.** Options that return a tool result as prose, put it under the wrong role, or drop the ID all lose the link the protocol relies on. Ask what breaks when **two tool calls are open at once**.
- **A non-technical constraint postponed.** Contracts, data residency and billing come from outside engineering and get treated as something to schedule. They **apply from the first call**.

Match on what the scenario is doing, not the industry it's set in:

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| Thousands of items, no deadline on any single result, cost matters | A batch, with results matched afterwards | A synchronous loop, or a bigger token limit offered as throughput |
| Users should see output as it's produced | Streaming | Caching, a bigger limit or a schema, each answering a different need |
| A tool ran and the model needs its output | A user turn of `tool_result` blocks with the call's ID | Prose in a user message, the result in the assistant turn, or the system prompt |
| An answer stopped short and nobody knows why | Read `stop_reason` and do what it requires | Treating HTTP `200` as proof the content is complete |
| A contract or compliance rule names a cloud provider | Use that platform now, behind one provider-specific layer | Building direct and migrating later, or maintaining both |
| The same large document is sent on every turn | Upload it once and refer to it by ID | Re-encoding the file into each request |
| Batch results don't match the order you sent them in | Join on the `custom_id` you assigned | Assuming results come back in order |
| Code prefills the start of Claude's answer on a current model | Remove the prefill; use structured outputs or instructions | Keeping the prefill, which now returns a `400` |

One habit does more work than the table. **Name who's waiting** before looking at any option. Then ask whether the option changes **delivery**, **cost**, **shape** or **capability**. Most wrong answers here are correct answers to one of the other three.

## What to Remember

- The **Messages API is stateless**: every call sends the **whole history**. A request has a **model**, a **`max_tokens`** limit, a **`messages`** list of role-and-content turns, and standing instructions in a top-level **`system`** field. **Message threads (beta)** can hold a conversation on the server for a few minutes to speed up long agent loops.
- **Content is a list of typed blocks**, which is why text, images, documents, tool calls and reasoning all use **one endpoint**. A response returns blocks, a **`stop_reason`** and a **`usage`** count.
- **Prefilling Claude's answer returns a `400`** on current models. Synthetic assistant turns *earlier* in the history are still fine.
- **The waiting party** (someone watching, a blocked caller, a caller doing other work, or nobody until a deadline) **chooses the endpoint** before you write the prompt.
- **Seven `stop_reason` values, four actions**: **nothing** (`end_turn`, `stop_sequence`), **continue** (`tool_use`, `pause_turn`), **mark incomplete** (`max_tokens`, `model_context_window_exceeded`), **retry elsewhere** (`refusal`, which arrives as a normal **HTTP `200`**).
- A **tool result is a user turn** of `tool_result` blocks, each **repeating the `tool_use_id`**, with **nothing else** in that message, the **`tools` array unchanged**, and the assistant content (including thinking) **sent back exactly as received**.
- **Streaming delivers the same message sooner**, not cheaper or faster overall. **`stop_reason` arrives in `message_delta`**, and an **error can arrive after HTTP `200`**, which the SDK won't retry.
- **Images and PDFs are content blocks**, sent as **base64**, a **URL** or a **`file_id`**. With **more than 20 images**, a stricter size limit applies to all of them. **PDF pages are billed as text and as images**, so page count drives the cost.
- **Batches cost 50% less**, usually finish **within an hour**, **expire at 24 hours**, and hold up to **100,000 requests**. Results come back **in any order**, matched by **`custom_id`**. `errored`, `canceled` and `expired` results **aren't billed**. `stream`, fast mode and fallbacks are **rejected** in a batch.
- **Concurrent requests aren't batching.** Concurrency keeps your process busy while answers still arrive at normal speed. A batch gives up that speed for the discount.
- **Cloud providers keep the message shape** but change **authentication, endpoints, model IDs, billing and feature availability**. On Google Cloud the **model goes in the URL** and **`anthropic_version` goes in the body**.

## Endnotes for this chapter

1. The Messages API is stateless, so the full conversation history is sent each time; earlier turns can be synthetic assistant messages; the request takes a model, `max_tokens`, a `messages` list and a top-level `system` field; and the response carries a `content` array, `stop_reason` and `usage`. Anthropic, *API usage primer for Claude* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/claude_api_primer](https://platform.claude.com/docs/en/claude_api_primer)

2. A request to Claude Sonnet 5.5 leaves out five settings that return a 400 error, including assistant prefill. Anthropic, *Claude Sonnet 5.5 migration guide* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/models/sonnet-5-5/migration-guide](https://platform.claude.com/docs/en/models/sonnet-5-5/migration-guide)

3. A mid-conversation system message is appended with `role: "system"` where the instruction starts to apply, must immediately follow a user turn, and avoids editing the top-level system field, which sits near the start of the cached prefix. Anthropic, *Mid-conversation system messages and tool changes* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages](https://platform.claude.com/docs/en/build-with-claude/mid-conversation-system-messages)

4. With message threads, in beta, the API keeps the conversation so each later request sends only the new messages and the previous response's ID; a thread expires 5 minutes after its last request and is deleted no more than 1 hour after its last use; threads mainly lower time to first token in long agent loops. Anthropic, *Message threads* and *API usage primer for Claude* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/message-threads](https://platform.claude.com/docs/en/build-with-claude/message-threads)

5. A `tool_use` block carries `id`, `name` and `input`; the continuation is a user message of `tool_result` blocks, each carrying the `tool_use_id` of the call it answers, optional `content` and an optional `is_error` flag; further input is sent as a separate user message after the turn completes. Anthropic, *Handle tool calls* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)

6. The `stop_reason` values `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `pause_turn`, `refusal` and `model_context_window_exceeded`; text added right after a `tool_result` can produce an empty response; the `tools` array stays the same; `pause_turn` is returned when a server-tool loop hits its iteration limit (default 10) and is continued by appending the content unchanged; refusals arrive as HTTP 200 with `stop_details`, can often be retried on another model, Claude Haiku 5.5 has no server-side fallback, and a `reasoning_extraction` refusal should be fixed by changing the prompt; and `stop_reason` is null in `message_start` and provided only in `message_delta`. Anthropic, *Stop reasons and fallback* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)

7. Pass thinking blocks back unmodified in tool-use loops and keep the conversation append-only. Anthropic, *What's new in Claude Opus 5.5* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/models/opus-5-5/whats-new-opus-5-5](https://platform.claude.com/docs/en/models/opus-5-5/whats-new-opus-5-5)

8. Setting `"stream": true` returns the response over server-sent events, in the order `message_start`, `content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`, `message_stop`, with `ping` events interspersed; errors such as `overloaded_error` can arrive inside the stream where a non-streaming request would return 529; unknown event types should be handled gracefully; and the SDKs stream internally for requests with large `max_tokens` to avoid HTTP timeouts. Anthropic, *Streaming messages* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/streaming](https://platform.claude.com/docs/en/build-with-claude/streaming)

9. Over server-sent events, an error can occur after the API returns 200, and the SDK doesn't retry errors inside a stream. Anthropic, *Claude API errors* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/api/errors](https://platform.claude.com/docs/en/api/errors)

10. Images are sent as `image` blocks using base64, URL or file_id; JPEG, PNG, GIF and WebP are supported up to 8000 × 8000 pixels; multiple images are analysed together and earlier-turn images remain available; and with more than 20 image blocks a stricter per-image dimension limit applies to every image in the request, with oversized blocks rejected. Anthropic, *Vision* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/vision](https://platform.claude.com/docs/en/build-with-claude/vision)

11. PDFs are provided as a URL, base64 or a Files API reference; token cost combines extracted text with image tokens because each page is converted to an image; and size and page limits apply to the whole request. Anthropic, *PDF support* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/pdf-support](https://platform.claude.com/docs/en/build-with-claude/pdf-support)

12. The Files API lets you upload a file once, receive a `file_id`, and reference it in later requests instead of re-uploading. Anthropic, *Files API* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/files](https://platform.claude.com/docs/en/build-with-claude/files)

13. Feature availability differs by platform across the Claude API, Amazon Bedrock, Claude Platform on AWS, Google Cloud and Microsoft Foundry, with the Files API available on the Claude API and in beta on some platforms. Anthropic, *Features overview* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/overview](https://platform.claude.com/docs/en/build-with-claude/overview)

14. The Message Batches API bills at 50% of standard prices; most batches complete in under an hour and batches expire after 24 hours; a batch holds up to 100,000 requests or 256 MB; `custom_id` matches `^[a-zA-Z0-9_-]{1,64}$` and results return in any order; `succeeded` is billed while `errored`, `canceled` and `expired` are not; a failed request doesn't affect others; results are downloadable for 29 days from creation; batches can be cancelled and may contain partial results; `stream`, `speed`, `fallbacks` and `max_tokens: 0` are rejected; and caching stacks with the discount, with best-effort hits of typically 30%–98% and the 1-hour duration recommended. Anthropic, *Batch processing* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/batch-processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)

15. Claude on Google Cloud uses nearly the same API as the Messages API, except that `model` is given in the endpoint URL and `anthropic_version` is passed in the body; Anthropic's SDKs support the platform. Anthropic, *Claude on Google Cloud* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai](https://platform.claude.com/docs/en/build-with-claude/claude-on-vertex-ai)

16. Claude in Amazon Bedrock runs on AWS-managed infrastructure inside the AWS security boundary and uses the same Messages API shape. Anthropic, *Claude in Amazon Bedrock* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock)

17. Claude in Microsoft Foundry bills usage through the Azure Marketplace and manages cost through the Azure subscription. Anthropic, *Claude in Microsoft Foundry* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry](https://platform.claude.com/docs/en/build-with-claude/claude-in-microsoft-foundry)

18. Fast mode for Claude Opus 5 was available on the Claude API only, not on Amazon Bedrock, Claude Platform on AWS, Google Cloud or Microsoft Foundry. Anthropic, *What's new in Claude Opus 5* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/models/whats-new-opus-5](https://platform.claude.com/docs/en/about-claude/models/whats-new-opus-5)
