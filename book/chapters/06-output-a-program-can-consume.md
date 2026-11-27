# Chapter 6: Valid Format Doesn't Mean Correct

**Summary**: *A schema tells you the response has the right fields with the right types. It says nothing about whether the numbers in those fields are true. "Well formed, confidently worded and wrong" is a common state, and no validator can tell it apart from "well formed and right". This chapter covers four steps that belong between the model and your database: fixing the format during generation, checking the response when it arrives, parsing it so surprises don't break things, and verifying the content against something the model didn't produce.*

## A Response Can Pass Every Rule You Wrote and Still Be Wrong

Picture a reporting pipeline that pulls figures out of financial filings into a ledger. Responses come back as JSON, **every field present**, **every type correct**, each number stated calmly and specifically. Nothing raises an error for weeks. Then a **reconciliation** finds several figures that were **plausible, well formed and wrong**.

Look for the check that should have caught them. Every check that existed was **checking format**: the validator asked whether `revenue` was a number, and it was. **Nothing asked whether it was the right number.**

That gap deserves a name. Every check answers one of two questions:

- **The format question**: are the fields present, are the types right, are the values in the allowed set?
- **The truth question**: does the content match the real world?

Constrained generation, schema validation and defensive parsing answer the first question completely and cheaply. **None of them touches the second.**

**Treating validity as verification** is the mistake, and it survives review because the green checkmark is real. Something genuinely was verified. It was just the format.

## A Prompt Asks for a Format; a Grammar Removes the Alternatives

Chapter 5 covered asking for a format in the prompt, and that works most of the time. A prompt instruction holds on the inputs you tested and slips on the one you didn't.

**Structured outputs** close that gap in the API instead of in the wording. You supply a **JSON Schema**. The API compiles it into a **grammar** that limits what Claude can generate, so a response that **breaks the schema can't be produced at all**. It's generally available on all models with no beta header.<sup>[1]</sup> There are two parts, and you can use either or both in one request:<sup>[1]</sup>

- **`output_config.format`** with `type: "json_schema"` controls the **final response**.
- **`strict: true`** on a tool definition guarantees valid **tool names and inputs**. Chapter 9 covers that half.

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Extract the invoice fields: ..."}],
    output_config={"format": {"type": "json_schema", "schema": {
        "type": "object",
        "properties": {
            "invoice_number": {"type": "string"},
            "total": {"type": "number"},
            "currency": {"type": "string", "enum": ["EUR", "USD", "GEL"]}
        },
        "required": ["invoice_number", "total", "currency"],
        "additionalProperties": False
    }}},
)
```

The difference from a prompt instruction isn't one of degree. **Constraining generation removes the failure entirely**; making the parser more robust only changes **how loudly the consumer fails**.

The guarantee has costs, and they all come from the grammar:<sup>[1]</sup>

- **Compilation latency** the first time a schema is used. Compiled grammars are cached for **24 hours** from last use, so a stable schema is cheaper than one rebuilt per customer.
- **An extra system prompt** describing the format is added, which costs input tokens.
- **Changing `output_config.format` invalidates the prompt cache** for that conversation.
- **Complexity limits**: at most **20 strict tools**, **24 optional parameters** and **16 union-type parameters** per request. Beyond internal limits you get a `400` ("Schema is too complex for compilation"). Optional fields add the most complexity, so make fields required where you can.

Two compatibility rules matter:<sup>[1]</sup>

- **Structured outputs and citations can't be combined.** Enabling both returns a `400`. You have to choose between a guaranteed format and returned source passages in a single request (more on citations below).
- The grammar applies only to Claude's **final output**, not to thinking or tool calls, so Claude can still reason freely before producing the constrained answer.

**Prefilling** the assistant turn was the old way to force a format. Structured outputs are its replacement on Claude 4.6 and later, where prefill returns a `400`.<sup>[2]</sup>

Agents get the same mechanism one level up. The **Agent SDK** validates the agent's result against your schema, **re-prompts on a mismatch**, and gives up with the subtype `error_max_structured_output_retries`. A quieter failure sits next to it: a run can end with subtype **`success`** and **no `structured_output`** at all, and that should be treated as a failure too.<sup>[3]</sup>

## Three Documented Ways a "Guaranteed" Schema Gives You Something Else

Constrained decoding guarantees what's *generated*, not every response that reaches your parser. Three documented exceptions arrive as an **ordinary HTTP `200`**:<sup>[1]</sup>

| The exception | What your code should do |
|---|---|
| A **refusal** (`stop_reason: "refusal"`). It's billed, and the output may not match the schema | Check `stop_reason` before parsing; retry on another model where appropriate<sup>[4]</sup> |
| The **token limit** (`stop_reason: "max_tokens"`), with the JSON cut off mid-object | Raise `max_tokens` and retry. Don't try to salvage the fragment |
| An **enum value in different capitalisation**, such as "Conversation Topic 3" for "Conversation topic 3", with no error and no special stop reason | Compare enum values case-insensitively, and never define two that differ only in case |

The last one makes this chapter's point twice: **nothing reports anything**, and a value your code switches on **matches no branch**.

One more trap: asking for a step-by-step **reasoning field** in your schema can trigger a `reasoning_extraction` refusal. Ask for a short explanation field instead.<sup>[1]</sup>

## Validating on Arrival Needs Somewhere to Send Failures

**Validating** means checking the response against the schema in your own code, when it arrives, before anything downstream sees it. That sounds redundant, but it isn't. The SDK helpers **remove constraints** that the grammar doesn't support, such as `minimum` and `maxLength`, and move them into the field's description. So Claude sees a simplified schema, and only your validation enforces the original rules.<sup>[1]</sup>

> **Example.** Your schema says `"age": {"type": "integer", "minimum": 0, "maximum": 120}`. The SDK sends Claude `"age": {"type": "integer"}` with the range mentioned in the description. Claude can still produce `150`, and the grammar allows it. Only your validation on arrival catches it.

**Where the failure goes** matters more than the check, because a check with nowhere to send failures is just a crash with extra steps. Choose one of three: **retry** the call, **reject** the record into a queue someone works through, or **route** it to a person.

Choosing between them needs **categories**, not a single number. A pipeline that routes everything on one **global confidence threshold** has squashed three different situations onto one scale:

- An **optional field that's genuinely missing** (fine, store it as absent).
- A **required field too poor to read** (send it to a person).
- A **document that was never in scope** (reject it).

No threshold separates these. Name the three outcomes, give an example of each, and route on the category.

## Defensive Parsing Survives Surprises and Still Reports Them

Two failures sit on either side of this:

- **Crashing on anything unexpected** is the outage you were asked to prevent.
- **Silently dropping whatever doesn't fit** is worse. The pipeline reports success while records quietly disappear. This **silent drop** is the named mistake, and it's tempting because it makes the alerts stop.

Parsing gracefully means doing both halves: the odd case **survives**, and every rejection leaves a **log line, a metric and the payload**, so the rejection rate is a number someone can see.

| The surprise | How to handle it |
|---|---|
| A field you didn't expect | Ignore it, but log that it arrived. A new field is how a producer signals a change |
| A missing value | Keep **absent**, **null** and **empty** as three different states. Merging them invents information |
| A type that doesn't match | Convert when the conversion is exact; reject when it's a guess. A string that fails to become a number is a rejection, **never a zero** |

> **Example.** A response has `"discount": ""`. If your parser turns that into `0`, you've just recorded that the customer got no discount, which nobody said. Rejecting it, or storing it as "empty", keeps the truth.

The same rule applies in the other direction, where your code feeds the model. A tool that **swallows its own failure** and returns an empty result gets read as valid data, and the model reasons confidently about nothing. Mark the failure (`is_error: true`) so the model knows. And in your own code, catch the SDK's **typed exception classes** rather than matching error text.<sup>[5]</sup>

## A Check Only Counts if Its Truth Comes From Outside the Model

Now for the **truth question**. The tempting answers all share one flaw:

- **Ask the model for a confidence score.** One generation rates another, with no new information in between. The score tends to be highest exactly where the answer is most confidently wrong, which is the case you needed it for.
- **Sample several times and take the majority.** A consistent error wins the vote just as easily as a right answer. Anthropic documents **best-of-N** as a way to spot **inconsistency**, which can signal a problem.<sup>[6]</sup> But **agreement isn't the opposite signal**, even though it's constantly read as one.
- **Switch to a stronger model.** The error rate falls, but the *kind* of failure stays, and nothing downstream detects it.

This book calls all of these **correlated verification**. The rule underneath is short: **a check catches a confident wrong answer only if its own correctness comes from something outside the model.**

That points to unglamorous work, and the unglamorous work is the answer:

- **Recompute derived values** in code and compare. Arithmetic is the cheapest independent authority you have.
- **Cross-check totals and IDs** against the source records.
- **Enforce range and consistency rules** that come from your domain, not from the response.
- **Send every violation to human review**, instead of retrying until the check passes.

> **Example.** An extraction returns `{"subtotal": 1200, "tax": 240, "total": 1400}`. The schema is happy. One line of code, `abs(subtotal + tax - total) < 0.01`, catches that 1200 + 240 is 1440, not 1400. No amount of re-asking the model would have been as reliable.

Two documented techniques shrink the problem up front. **Restricting Claude to the supplied documents**, instead of its general knowledge, removes claims that have no source at all.<sup>[6]</sup> **Citations** return the exact passages that support each claim, so a person or a program can check them.<sup>[7]</sup> Remember that citations can't be combined with structured outputs in the same request.<sup>[1]</sup>

**Provenance belongs in the schema**, for the same reason the figures do. Someone acting on a claim needs its support as fields that answer four questions:

| Field | Question it answers |
|---|---|
| `source_type` | What kind of source was it? |
| `excerpt` | Which passage or returned value does it rest on? |
| `observed_at` | When was it observed? |
| `verified` | Has anyone checked it? |

Compressing that into a neat paragraph removes exactly what the reviewer needs. Attaching the raw transcript instead makes every reviewer redo the same reconstruction.

One mirror case is worth remembering. When **two credible sources disagree**, a single confident answer gets **invented, not found**. Averaging creates a third figure nobody reported. Keeping the larger one turns a preference into a "finding". Searching until something supports one side is a hunt with a fixed verdict. Instead, **keep both, attributed, with the conflict marked**. An output that shows the disagreement is the accurate one.

## How the Exam Builds Wrong Answers

Wrong answers here describe real engineering, and someone has shipped every one of them. Four assumptions produce most of them:

- **A format fix offered for a truth problem.** The output validated and was still wrong, and the option adds a schema, a stricter validator or a better prompt. All three fix a parsing problem nobody had.
- **A check that comes from the same process it's checking.** Self-reported confidence, majority vote, self-grading, a bigger model. Each one fails exactly where the answer fails.
- **Asking for cooperation when enforcement is available.** Asking for JSON, asking more firmly, retrying until something parses. None of these fixes the boundary, and none detects being ignored.
- **Hiding the failure instead of fixing it.** Dropping non-conforming output quietly, turning a missing value into a present one, deleting the source that disagrees. The symptom stops, and nothing has improved.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| Output passes validation and is wrong anyway | Recompute or cross-check outside the model, and send violations to a person | A stronger model, a stricter schema, or a model-filled confidence field |
| The format varies between calls and a program reads it | Constrain generation with structured outputs, then validate on arrival | A regex over today's format, or a firmer prompt |
| Code crashes on an unexpected field, missing value or wrong type | Parse defensively: survive it, and keep it visible | Silently dropping non-conforming output, or leaving the crash |
| Credible sources disagree, or a claim must support someone else's decision | Keep the conflict and provenance as fields: source, excerpt, time, verification status | Averaging, keeping the larger value, or a narrative the reviewer must untangle |
| Extraction routes everything on one confidence threshold | Name and demonstrate the outcomes: missing-optional, unreadable-required, out-of-scope | One global threshold, or retrying until every field is filled |
| The response is refused, truncated or oddly capitalised | Check `stop_reason` before parsing; compare enums case-insensitively | Treating any `200` as a successful result |
| The team wants a guaranteed JSON format *and* cited source passages in one call | Two steps, or provenance fields in the schema; the two features can't be combined in one request | Enabling both, which returns a `400` |

One habit works beyond the table. For any option offered as a **reliability fix**, ask: what would it take for the output to be **wrong and still pass** this check? If nothing outside the model was consulted, it only answers the format question.

## What to Remember

- Every check answers either the **format question** (fields, types, allowed values) or the **truth question** (does the content match the world). Schema validation, constrained generation and defensive parsing only answer the first.
- **Structured outputs** compile your JSON Schema into a **grammar**, so output that breaks the schema **can't be generated**. `output_config.format` controls the final response; `strict: true` controls tool names and inputs. It's **GA on all models**.
- **Constraining generation removes the failure**; parsing harder only changes how loudly it fails. The grammar costs **first-use latency**, an **extra system prompt**, and has **complexity limits** (20 strict tools, 24 optional parameters, 16 union types). **Structured outputs and citations can't be combined.**
- **Three documented exceptions** return `200` without a usable structure: a **refusal**, a **`max_tokens` cut-off**, and an **enum in different capitalisation** with no error. Check `stop_reason` first, and compare enums case-insensitively.
- **SDKs strip range and length constraints** from the schema they send, so your own **validation on arrival** enforces them. A failed validation needs **somewhere to go**: retry, reject to a queue, or send to a person, chosen by **named category**, not a single threshold.
- **Defensive parsing survives surprises and reports them**: ignore unknown fields but log them, keep **absent, null and empty** separate, reject a type mismatch rather than guessing, and **log every rejection**. The **silent drop** is the worse failure.
- A check **catches a confident wrong answer** only if its correctness comes from **outside the model**. Self-reported confidence, majority vote and a bigger model all fail exactly where the answer fails.
- Independent verification is **arithmetic and lookups**: recompute derived values, cross-check against source records, enforce domain rules, and send violations to people. **Provenance is a set of fields.** Conflicting sources are kept **with the conflict marked**, because averaging invents a number nobody reported.

## Endnotes for this chapter

1. Structured outputs compile a JSON Schema into a grammar for constrained decoding; `output_config.format` controls the final response and `strict: true` validates tool names and inputs, usable together; the feature is GA on all models and platforms of the Messages API; SDK helpers remove unsupported constraints such as `minimum` and `maxLength`, move them into the field description, add `additionalProperties: false`, and validate against the original schema; the first use of a schema adds compilation latency, compiled grammars are cached for 24 hours, an extra system prompt is injected, and changing `output_config.format` invalidates the prompt cache; limits are 20 strict tools, 24 optional parameters and 16 union-type parameters, with a 400 beyond internal limits; compliance is not guaranteed on refusals, on `max_tokens` truncation, or for enum and const capitalisation, which completes with no error; requests for step-by-step reasoning fields can trigger a `reasoning_extraction` refusal; grammar constraints apply only to Claude's direct output; and structured outputs are incompatible with citations (400) and with prefilling. Anthropic, *Structured outputs* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/structured-outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)

2. Claude 4.6 and later models do not support prefilling the last assistant message, which returns a 400; the alternatives are structured outputs, system prompt instructions or `output_config.format`. Anthropic, *Claude API errors* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/api/errors](https://platform.claude.com/docs/en/api/errors)

3. The Agent SDK validates agent output against a JSON Schema, re-prompts on a mismatch, and returns `error_max_structured_output_retries` when the retry limit is reached; a result can also end with subtype `success` and no `structured_output`, which should be treated as a failure. Anthropic, *Get structured output from agents* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/structured-outputs](https://code.claude.com/docs/en/agent-sdk/structured-outputs)

4. Refusals arrive as HTTP 200 with `stop_reason: "refusal"` and `stop_details`; you can often get a response by retrying on another model. Anthropic, *Stop reasons and fallback* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons](https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons)

5. The SDKs raise typed exceptions, and the guidance is to catch the typed classes rather than match on message text. Anthropic, *Claude API errors* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/api/errors](https://platform.claude.com/docs/en/api/errors)

6. Techniques for reducing hallucinations include best-of-N verification, where inconsistencies across outputs could indicate hallucinations, and restricting Claude to information in the provided documents. Anthropic, *Reduce hallucinations* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)

7. Citations ground responses in supplied documents and return the exact passages that support each claim. Anthropic, *Citations* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/citations](https://platform.claude.com/docs/en/build-with-claude/citations)
