# Chapter 19: A Test Says It Broke; a Trace Says Where

**Summary**: *This chapter is about four "where" questions. Where an error's cause is, which decides whether waiting will help. Where in the run a failure happened, which only a step-by-step record can show. Where in the stack it started (your integration code or the model's output), which is what most scenarios turn on. And where in a population a measurement holds, which decides what an average can tell you. Get the location wrong and every fix after it targets the wrong thing.*

## A Passing Suite Is a Claim About Parts, Not the Product

The **retrieval function passes** its unit tests. So does the **prompt builder**, and a check confirms the model call returns the fields the parser wants. But run the flow end to end and it answers a policy question from **general knowledge** instead of the **retrieved policy**, with **no error anywhere**.

The **join** broke. Retrieval returns a **list of records**; the builder expects **text**; the prompt ends up containing something the model reads as noise. Both components are correct. **Nobody wrote the contract** between them, so nothing tested it.

Every check sees a **limited field of view**: one function, one call, one flow, one number. A failure can only be found by a check that can see it. A **seam belongs to neither component**, so nothing sees it until you build a check **aimed at the seam**.

> **Example.** An integration test that feeds the prompt builder the **real output** of the retrieval function (a list of records, not a hand-made string) and asserts that the resulting prompt contains the policy text would have caught this in seconds.

The anti-pattern is **coverage counted by component**. Every part is covered. The **joins aren't parts**.

## An Error's Category Is Where Its Cause Is, Not Its Status Code

One question sorts every failure: **could waiting alone change the outcome?**

- **Yes**: the cause is the **state of the world**: capacity, load, a per-minute limit, a dropped connection. Typical shapes: **`429`** rate limits, **`529`** overloads, **5xx** server errors and **timeouts**.<sup>[1]</sup> **Waiting is a real fix** for these, and these only.
- **No**: the cause is the **request**: a malformed body, a revoked key, a missing permission, a resource that doesn't exist (**`400`, `401`, `403`, `404`**).<sup>[1]</sup> Sending it again returns the **same error**. **Fix it or reject it**; don't retry.

The status code is only a **first guess**. Documented cases break the mapping both ways:

- A **`429` from hitting your usage tier's monthly spend cap** has **no `retry-after` header** and keeps failing until access resumes.<sup>[1]</sup> Same status, other category; the **missing header is the clue**.
- A **spend limit you set** on your organisation or workspace returns a **`400`**, not a `429` (except Claude Code workspace limits, which may return `429`).<sup>[1]</sup>
- A request **too large for the context window** won't succeed if sent unchanged. Claude Code **reduces `max_tokens`** and sends a different request.<sup>[2]</sup>
- On a **streaming** request, an error can arrive **after a `200` response** has started, so status-code handling alone misses it.<sup>[1]</sup>

The official SDKs **already retry** connection errors, rate limits and 5xx errors with exponential backoff, **twice by default**, honouring `retry-after`; `max_retries` changes or disables that.<sup>[1]</sup> The anti-pattern is **reading the status code as the diagnosis**. The code suggests a cause; the **header, message and request ID** confirm it.

## Misclassifying Costs Silence One Way and Noise the Other

The two mistakes cost differently:

- **Treating a permanent failure as temporary**: every retry is **guaranteed to fail**. Your budget drains, the real error shows up late looking like slowness, and on a rate limit each retry **makes it worse**. This can run for **months**.
- **Treating a temporary failure as permanent**: something that would have cleared in seconds reaches a user as a **broken feature**. It gets **reported the same day**.

So when the cause is unclear, **treat it as permanent**: it's the cheaper mistake.

There's a second, separate question: **has anything already happened?** Whether to retry depends on the **cause**. Whether it's **safe** to retry depends on what the first attempt **already did**. Claude Code won't re-send a request that failed **after Claude completed a tool call**, because that would **run the tool calls twice**.<sup>[2]</sup> Any retry past a step that **writes** is a second write, unless the write is **idempotent**.

> **Example.** An agent calls `create_refund`, the tool succeeds, and then the connection drops before Claude's next message arrives. Re-sending the whole request would ask Claude to decide again and could issue a **second refund**. The safe approach: keep the completed tool result, continue from there, and give `create_refund` an idempotency key so a duplicate call is ignored.

Retry mechanics are covered in chapter 2. When deciding who to alert, **impact and recoverability** predict whether a person must act now; a single severity label for every error predicts nothing.

## A Trace Records Each Step, and You Build It Before the Failure

A **test tells you** a failure exists. It doesn't tell you **which step** caused it. A **trace** does: it records every step of the run with its **prompt**, **tool calls**, **intermediate output** and **timing**, so the failing step is visible instead of guessed.

The Agent SDK's OpenTelemetry support shows the shape:<sup>[3]</sup>

- Each **turn** of the agent loop is a span.
- Each **API call** is a child span with **model name, latency and token counts**.
- Each **tool call** is a child span, with **waiting for permission** separated from **execution**.
- A **subagent's spans nest** under the parent's tool span, so a delegation chain reads as one trace.

Two properties decide what that record is good for:<sup>[3]</sup>

- **Telemetry is structural by default.** It records durations, model names and tool names, but **not the content** the agent reads and writes; content is **opt-in**. The default trace tells you which step and how long, not what was said.
- **Export fails silently.** If the collector is unreachable, the agent keeps running and the telemetry is **dropped**, with no error in your application. Monitor the monitor.

## Read the Model's Output Before Blaming Either Side

Open the trace and **read what the model returned**, before anything downstream touched it. There are three possibilities:

| What the trace shows | Where the fault is | What to change |
|---|---|---|
| The output is what you wanted, but the system did the wrong thing with it | **Below the model** | The parser, handoff, tool wrapper or dispatcher |
| The output is wrong, and the **input** was truncated, malformed or missing its retrieved context | **Still below the model** | The step that built the input |
| The output is wrong, and the input was **complete and well formed** | **The model's output** | Examples, constraints, prompt, model choice |

The **middle row costs teams weeks**. It looks like a model failure, and every fix aimed at the model leaves it in place. So the rule: it's a **model failure only after you've read the input the model actually received**. Read what was **sent**, not the template that was supposed to produce it.

> **Example.** A support bot gives wrong refund amounts. The team spends a week rewriting the prompt. The trace shows the order lookup tool's result was cut off at 4,000 characters by a wrapper, so the line items the model needed never arrived. The fix is in the wrapper, and the original prompt was fine.

Configuration debugging works the same way: first confirm the instruction **reached the context window** (in Claude Code, `/context` shows what's loaded). If it did, the problem is **how it's written**.<sup>[4]</sup>

A failure crossing a boundary **keeps its identity or loses it**. Chapter 9 covers tool results, where **`is_error`** tells the model a result is a failure, not data.<sup>[5]</sup> The general rule: one message that stands for an **invalid ID**, an **expired token**, a **timeout** and a **policy refusal** has thrown away the differences recovery depends on. **Recovery can only be as specific as the error that reaches it.**

## An Average Is Only True of the Population It Averaged

An **eval** looks at the **whole run's output**. It can't tell you **where** something broke. It answers a narrower question nothing else answers: **did your change help**, on a **fixed set of cases**?

**Build for coverage before polishing the rubric**:<sup>[6]</sup>

- **20 to 50 tasks** drawn from **real failures** is a good start.
- Include cases where a behaviour **should** happen and cases where it **shouldn't**.
- A good task is one where **two domain experts** would independently give the same pass or fail. **Ambiguity in a task becomes noise** in the metric.
- A **model judge** gives a confident-looking number that means nothing until you've **measured its agreement with expert human labels**, and kept recalibrating it. Read the transcripts and grades yourself.

Then **granularity**. An **overall score tells you nothing** about any group inside it. To slice by category, **the record must carry the category**: you can't measure along an axis the data doesn't hold. So when performance seems to differ by group, **adding that field comes first**. A flat average can also hide a change that **fixed one group and broke another**, which is why you read **per-case results**, not just the mean.

> **Example.** A ticket classifier scores 91% before and 91% after a prompt change. Broken down by category, billing went from 80% to 95%, and security went from 97% to 70%. The average said "no change". The per-category view says "ship the billing fix, revert the security regression".

## How the Exam Builds Wrong Answers

Wrong answers here are competent engineering aimed **one step too early** or **one level too coarse**. Four assumptions produce most of them.

- **A fix applied before the cause is found.** Swapping the model, raising a threshold, adding a validator, tweaking a setting. Each is a real fix offered while the **failing step is unknown**. It's right only once the scenario has isolated the break.
- **A distinction collapsed instead of restored.** One severity for every error, one message for every backend fault, one confidence cutoff for every outcome, an empty result standing in for a failure. The scenario usually names the collapse as the symptom; the option offering more of it looks like tidying.
- **An aggregate where a segment was named.** A global threshold, one overall accuracy figure, review spread evenly across categories. All fine when the **population is uniform**, which the scenario has usually just said it isn't.
- **Repetition where the request must change.** Retrying a rejection, duplicating requests against a rate limit, retrying without limit, adding a new credential to beat a throttle. Right only when **time** was the thing that could change.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| A status shows a temporary rate or capacity problem | Back off with a cap, honouring `retry-after` | A new credential, duplicate requests, or removing the check |
| A `429` has no `retry-after` and never clears | Treat it as a spend cap: stop and alert, don't retry | Retrying with longer backoff |
| The same request sometimes succeeds and sometimes fails | Capture full input and output for both, then compare | Changing a parameter, or calling it expected |
| Every component passes its tests but the flow is wrong | Test the handoff with real output from the producing step | More component tests, or blaming the model |
| The model "gets it wrong" but nobody has looked at the trace | Read the input the model actually received | Rewriting the prompt or switching models |
| One overall score is quoted and errors cluster by category | Record the category, measure per category, sample within clusters | A global threshold, or reviewing only low-confidence items |
| Part of a job failed and the rest succeeded | Report the failed part: what was tried, the error type, any partial result | Returning empty, aborting everything, or retrying without limit |

One habit works beyond the table: for any proposed fix, ask what it would need to **observe** to be right, then check whether **anyone in the scenario is observing it**.

## What to Remember

- Every check has a **field of view**. A **seam belongs to neither component**, so all-green component tests prove things about **parts, not the product**. Test the seams with real output.
- Ask: **could waiting change the outcome?** Yes → **`429`, `529`, 5xx, timeouts**: back off. No → **`400`, `401`, `403`, `404`**: fix or reject.
- **Status codes are a first guess.** A **spend-cap `429` has no `retry-after`** and never clears on its own; a spend limit **you** set returns **`400`**; streaming errors can arrive **after a `200`**.
- SDKs **retry transient errors twice by default**, honouring `retry-after`.
- **Permanent treated as temporary** drains budget silently for months; **temporary treated as permanent** gets reported the same day. **When unsure, treat it as permanent.**
- **Retriable** is about the **cause**; **safe** is about what **already happened**. A retry after a write is a **second write** unless it's **idempotent**.
- A **trace** records each step's **prompt, tool calls, output and timing**. Default telemetry is **structural, no content**; export **fails silently**.
- **Read what the model returned and what it received.** Right output, wrong behaviour → integration. Wrong output from bad input → integration. Wrong output from good input → prompt, examples or model.
- **Recovery is only as specific as the error that reaches it.** Don't collapse error types into one message.
- An **eval** shows whether a change helped, not where it broke. Start with **20–50 cases from real failures**, include **should-not** cases, and **calibrate any model judge** against human labels.
- **Averages hide segments.** **Record the category**, measure per category, and **read per-case results**.

## Endnotes for this chapter

1. The Claude API returns `400` `invalid_request_error`, `401` `authentication_error`, `402` `billing_error`, `403` `permission_error`, `404` `not_found_error`, `409` `conflict_error`, `413` `request_too_large`, `429` `rate_limit_error`, `500` `api_error`, `504` `timeout_error` and `529` `overloaded_error`. A `429` can mean a rate limit, the usage tier's monthly spend cap, or a Claude Code workspace spend limit; a tier spend-cap `429` has no `retry-after` header and keeps failing until access resumes. Reaching a spend limit you set for an organisation or workspace returns a `400`, except on a Claude Code workspace. The official SDKs retry transient failures (connection errors, rate limits, 5xx) with exponential backoff, twice by default, honouring `retry-after`, configurable with `max_retries`. With streaming, errors can occur after a `200` response has begun. Every response carries a `request-id` header, also returned as `request_id` in error bodies. Anthropic, *Claude API errors* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/api/errors](https://platform.claude.com/docs/en/api/errors)

2. Claude Code retries transient failures up to 10 times with exponential backoff; when input plus `max_tokens` exceeds the context limit, it retries with a reduced `max_tokens`; it does not retry a server error, dropped connection or stalled stream arriving after Claude has completed a block of text or a tool call, because re-running could execute the same tool calls twice, and instead keeps what was completed and continues. Anthropic, *Error reference* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/errors](https://code.claude.com/docs/en/errors)

3. With enhanced telemetry, each agent loop turn is a `claude_code.interaction` span; each API call is a `claude_code.llm_request` span with model name, latency and token counts; each tool call is a `claude_code.tool` span with child spans for permission wait and execution; subagent spans nest under the parent's tool span. Telemetry is structural by default, and content is recorded only with opt-in variables; export errors fail silently, dropping telemetry with no error in the application. Anthropic, *Observability with OpenTelemetry* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/observability](https://code.claude.com/docs/en/agent-sdk/observability)

4. `/context` shows everything in the context window, including memory files and skills, and is the first check on whether an instruction loaded; if it loaded but isn't followed, the issue is likely how it's written. Anthropic, *Debug your configuration* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/debug-your-config](https://code.claude.com/docs/en/debug-your-config)

5. A `tool_result` block has an optional `is_error` field, set to true when the tool execution failed. Anthropic, *Handle tool calls* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)

6. 20 to 50 simple tasks drawn from real failures is a good start; test both cases where a behaviour should occur and where it shouldn't; a good task is one where two domain experts would independently agree on pass or fail; ambiguity becomes noise in metrics; model-based rubrics should be calibrated against expert human judgement, and graders can't be trusted without reading transcripts and grades from many trials. Anthropic, *Demystifying evals for AI agents* (Anthropic Engineering, accessed 2026-09-09; not re-checked for this edition), [https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
