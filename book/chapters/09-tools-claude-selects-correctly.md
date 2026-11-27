# Chapter 9: The Tool Description Is Input, Not Documentation

**Summary**: *A tool name is three words with no edges. When a request could fit either of two tools, Claude decides based on the text you wrote. Text written as documentation for a colleague usually leaves out the one thing that decides the choice: which requests belong to the tool next to it. This chapter covers the tool-use exchange, descriptions, schemas, checking arguments before you run them, errors Claude can act on, and why a tool set that grew without a plan costs you twice on every request.*

## One Request, Two Tools, and Nothing to Tell Them Apart

"Pull up everything on Hartley."

Two tools are registered:

- `search_customers` takes a name or account number and returns **profile records**.
- `search_orders` takes a customer or order ID and returns **line items and delivery status**.

Nothing in the request decides between them. A person on a support desk would ask. Claude **can't ask**; it makes a call. A wrong call runs a query nobody wanted and returns something that looks like an answer.

So what was the choice based on? Not the request, which is ambiguous. It was based on the **tool definitions**, which the API turns into part of the system prompt on every request, from your **names, descriptions and schemas**.<sup>[1]</sup> That text isn't reference material read afterwards. It's the **whole basis of the choice**.

That gives the chapter its rule. Every description needs a **boundary clause**: a sentence naming the requests this tool **doesn't** handle, and which tool handles them instead.

> **Example.** A description with a boundary clause:
>
> "Search customer profiles by name, email or account number. Returns contact details, plan and account status. Use this to identify who a customer is. **For orders, deliveries or refunds, use `search_orders` instead.**"

## The Exchange Has Five Stages, and You Own Four

Claude **never runs anything itself**. It produces a structured request, your code runs the operation, and the result comes back as a new block.<sup>[1]</sup> Treat a `tool_use` block as a **proposal from a caller that can be wrong**. Every failure in this chapter is a stage where that proposal was trusted too much.

| Stage | Who acts | What you're responsible for | What goes wrong if you skip it |
|---|---|---|---|
| Definitions are sent with the request | You | Names, descriptions and schemas, on every call | The wrong tool, or context wasted on unused tools |
| The response has `stop_reason: "tool_use"` and `tool_use` blocks, each with an `id`, `name` and `input`<sup>[3]</sup> | Claude | Nothing yet | |
| You inspect the proposal | You | Check the arguments; require approval where the action can't be undone | A malformed call reaches a remote service |
| The operation runs | You | Retry briefly on failures that clear on their own | One timeout ends the whole run |
| The result goes back | You | One `tool_result` per call, with the matching `tool_use_id`, in the next user message, **results first**<sup>[3]</sup><sup>[4]</sup> | A 400 error, or a result attached to the wrong call |

**Server tools** (such as web search) skip the middle three stages. Anthropic runs them and returns `server_tool_use` blocks already resolved, so **you send no `tool_result`**. If the server-side loop hits its iteration limit, you get **`pause_turn`**, and you resend the conversation to continue.<sup>[1]</sup> The trade-off is control: **validation, approval and audit records** can only happen on a machine you own. Chapter 13 covers the loop around all this.

## Anything a Description Leaves Out, Claude Has to Guess

Anthropic's advice is to describe a tool the way you'd explain it to **someone new on the team**.<sup>[10]</sup> The useful version of that test works backwards: read your description, then list what a new colleague would **still have to ask**. Each of those questions gets answered anyway, at selection time, by surface resemblance.

| Left out | What Claude doesn't know | What it does instead |
|---|---|---|
| What comes back | Whether this tool even answers the request | Calls it, reads the result, then calls the other one too |
| Which IDs are accepted | Whether it has the right kind of key | Passes whatever ID it has, which may be the wrong kind |
| Where the tool stops | Whether the other tool fits better | Matches the request's wording against the tool name |

**Three or four sentences** is enough: what the tool **does**, **when** to use it, what it **returns**, and the **boundary clause**. Longer isn't safer. A trigger condition in the ninth sentence competes with eight sentences nobody needs while choosing.

Two structural habits help more than any adjective:

- **Namespacing**: give related tools a **shared prefix** by service or resource (`billing_get_invoice`, `billing_refund`), so the catalogue reads as groups instead of a flat list. Anthropic reports that even choosing prefix vs suffix changed tool-use evaluation scores.<sup>[10]</sup>
- **Don't point at history.** A boundary clause that refers to earlier turns ("as discussed above") **fails silently** once history is trimmed.

The mistake here is **treating the name as enough**, and it always shows up as a saving: "the name says it, and descriptions are billed every request." But the description carries **when the tool applies**, which a name can't. Moving that text to a separate file deletes it. Moving it to the system prompt costs the same and helps less.

## When a Step Must Happen, Don't Rely on Wording

Sometimes a step must happen, and no description can guarantee it. That's configuration, not writing.

`tool_choice` has four settings: **`auto`** (Claude decides; the default), **`any`** (must call some tool), **`tool`** (must call the named tool) and **`none`** (no tool calls).<sup>[2]</sup> On older models, naming the tool on the first request was the standard way to enforce a required first step.

**On newer models, forcing is gone.** On Opus 5.5, Sonnet 5.5 and the Fable models from 5.1, `tool_choice` set to `any` or `tool` returns a **`400`**, including on the token counting endpoint. Only `auto` and `none` work. Opus 5 and older still accept all four.<sup>[11]</sup> Anthropic's migration advice is to use `auto`, state in the prompt when the tool applies, and use **strict tool use** or **structured outputs** when you need schema-valid JSON.<sup>[11]</sup>

So on current models, a step that **must** happen needs enforcement in **your code**:

- If the step doesn't need Claude's judgement (such as loading the customer record), **run it yourself** before calling Claude, and pass the result in.
- If Claude must decide the arguments, **check the first response** in your loop. If it isn't the required tool call, don't continue; send it back with a clear message, and stop after a bounded number of tries.

> **Example.** A refund agent must always call `verify_identity` before anything else. With Opus 5 you could set `tool_choice: {"type": "tool", "name": "verify_identity"}`. With Opus 5.5 that's a 400. Instead, your dispatcher refuses every other tool until `verify_identity` has returned success, and returns `is_error: true` with "Identity not verified yet. Call verify_identity first."

A firmer instruction alone is still **advice**, and advice is what produces intermittent failures.

## The `required` List Can Be Wrong in Both Directions

The **description decides which tool**. The **schema decides what goes in it**. The wrong tool and a missing parameter are two different problems, fixed in two different places.

Marking **every field required** doesn't make calls more complete. It forces Claude to fill a slot it has **no information for**, and it will, because the contract says it must. Marking too few fields causes the opposite: calls arrive incomplete and your function **makes up a default**. "Required" should mean **the operation makes no sense without this field**, and nothing weaker.

> **Example.** `create_ticket` marks `customer_phone` as required. A customer writes in by email with no phone number. Claude still has to fill the field, so it invents something like `"000-000-0000"`. Make the field optional, and the problem disappears.

Constrain the rest where you can:

- A **small `enum`** removes a decision instead of describing it.
- A **parameter description that states the format** ("ISO 8601 date, e.g. 2026-11-27") settles the shape at the place where it's read.
- **`input_examples`** attaches valid example inputs to the definition, which helps with nested or format-sensitive inputs. Each example is checked against your schema, and an invalid one returns a **400**.<sup>[2]</sup>

When a malformed argument would break something, use **`strict: true`**. It limits generation to schema-valid output, so an integer field **arrives as an integer**, not as the string "42".<sup>[5]</sup> It supports a subset of JSON Schema: a `pattern` with a **backreference, lookaround or word boundary won't compile**.<sup>[4]</sup> Chapter 6 covers the limits, such as **20 strict tools per request**.

## Check the Proposal Before You Run It, Because That's Your Last Chance

Your **dispatcher** sits between a caller that can be wrong and a system that **can't undo a write**. Check the arguments against the schema there, and treat a failure as a **normal path**, not as an exception that ends the run.

Three alternatives give away the same control:

- **Loosening the schema** until nothing fails makes a remote service the first thing that ever checks your inputs.
- **Logging the problem and running it anyway** is worse: your code caught the fault and sent it on regardless.
- **Retrying the same request unchanged** repeats an unpredictable step, with no limit.

The dispatcher is also where **approval belongs**, because it's the last moment a person can see an action before it happens. The Agent SDK models this as a **callback that pauses execution** until it returns one of three answers: **allow**, allow **with modified input**, or **deny with a message** Claude reads and can act on.<sup>[9]</sup> That last one matters. A denial with a reason is something Claude can work around; a denial with **no explanation** is a gap it can only guess about.

Add approval for an **irreversible action**, a plan about to **start executing**, and a result **outside the expected range**.

One mechanical trap: **never compare the raw JSON string** of the arguments, because Unicode and slash escaping differ between model versions. **Parse the JSON** and compare the values.<sup>[4]</sup>

## An Error Claude Can Act On Beats Any Kind of Silence

When a tool fails, that's information. What Claude learns depends entirely on what your code returns, and three of the four common choices **say something false**.

| What you return | What Claude now believes | What happens next |
|---|---|---|
| A made-up placeholder record | A fact | It flows into everything downstream, unmarked |
| An empty success | The lookup ran and found nothing | Someone is told there are no records |
| Nothing; the tool is quietly removed | It still has a capability it doesn't | It reasons around a gap it can't name |
| `is_error: true` with a short statement of what failed<sup>[3]</sup> | A failure, described | It retries, works around it, or reports the problem |

Only the **last row is true**, and truth is what makes recovery possible, because recovery can only be as specific as the error. Anthropic's advice is to make errors **actionable**: say what would have been valid, rather than returning a code or a stack trace.<sup>[10]</sup>

> **Example.** Instead of `"Error 422"`, return `"order_id 'HART-17' not found. Order IDs look like 'ORD-' followed by 8 digits. To search by customer name, use search_customers."`

**Retry belongs underneath, not instead.** A timeout that clears on its own is handled **inside the tool**, with **backoff** and a **limited number of attempts**, and never reaches the conversation. Turning a failure into a fake success is the expensive mistake: a visible error stops a run, while an invented value keeps it going in the **wrong direction**.

## A Bloated Tool Set Costs You Twice on Every Request

**Selection gets worse as the catalogue grows**, and fastest where descriptions **overlap**. Measured guidance: **fifty tool definitions** can use **10,000 to 20,000 tokens**, and selection accuracy drops once more than **30 to 50** are loaded at once.<sup>[8]</sup> The second cost is quieter: definitions are **sent and billed on every request**, along with a tool-use system prompt the API adds, whether or not any tool is used.<sup>[6]</sup>

So a tool set is something you design, with a running bill. There are three ways to shrink it:

- **Consolidate what's really one thing.** Several operations on the **same resource** can become one tool with an **`action` parameter** (documented guidance).<sup>[2]</sup> The version that fails is the **catch-all**: one tool covering unrelated jobs, with the choice pushed into a free-text argument that no schema checks. The decision didn't go away; it moved somewhere with less structure.
- **Expose only what the task needs.** Narrowing the tools a workflow can see shrinks both the context and the number of candidates. Adding a tool for every variation does the opposite.
- **Load on demand.** **Tool search** keeps definitions out of context until Claude looks them up. It suits catalogues of roughly **20+ tools** where most sit idle, at the cost of one **extra round trip** the first time. With **fewer than about 10**, sending everything up front is faster.<sup>[7]</sup><sup>[8]</sup>

Tools can also change mid-conversation. Mid-conversation system messages can **add or remove tools** without editing the `tools` list, and, with a beta header, carry **full tool definitions**, so you don't lose the prompt cache.<sup>[11]</sup> Chapter 3 covers those messages; chapter 10 covers the MCP servers that supply large catalogues.

## How the Exam Builds Wrong Answers

Every wrong answer here describes something real developers do: sharper prompts, fewer tools, a router. Four assumptions make them wrong:

- **Advice where enforcement is needed.** An instruction telling Claude to include a field or choose more carefully. **Instructions are suggestions**; schemas, `strict` and checks in your code **bind**.
- **A fake success instead of a real failure.** Placeholder data, an empty result, or a tool removed without a word. Each turns a stoppable error into a wrong answer that spreads.
- **Moving the choice instead of removing it.** Merging two different tools into a generic one, routing on keywords, or moving descriptions into the system prompt. The decision is still there, with less structure around it.
- **Turning a dial that doesn't touch the mechanism.** Reordering the tools array because position looks like priority, or changing a sampling setting (which current models reject anyway).

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| Two tools get confused, and their descriptions are short or auto-generated | Rewrite each to say when it applies, with a boundary against the other | Merging them, routing around them, or asking Claude to be careful |
| Required parameters are missing from calls | Tighten the schema: `required`, enums, format per parameter | A prompt instruction or a sampling setting |
| Calls arrive with arguments the schema doesn't allow | Validate in the dispatcher, as a known error path; or `strict: true` | Loosening the schema, or running the call after logging it |
| A tool times out now and then, and the run dies | Limited retry inside the tool, then a clear error | A made-up record, an empty success, or removing the tool |
| Definitions fill the context, and most go unused per task | Expose fewer tools per task, or load on demand with tool search | Treating the tool set as fixed |
| A specific step must happen first | On current models: run it in code, or have the dispatcher refuse other tools until it's done. On older models: force that tool with `tool_choice` | A firmer instruction, or forced `tool_choice` on Opus 5.5 (a 400) |
| Selection got worse after tools were added | Overlapping descriptions; consolidate and add boundary clauses | A hidden cap on tool count, or ordering read as ranking |

One habit works beyond the table. Ask **where the information an option changes is actually read**: in the description at selection time, in the schema at generation time, in your code at dispatch time, or in the result on the next turn. Most wrong answers apply a fix at the wrong one of those four points.

## What to Remember

- A **tool call is a proposal**, not trusted input. **Claude produces `tool_use`; your code runs it.** The result goes back as a **`tool_result` with the matching `tool_use_id`** in the next user message, one per call, **results first**.
- The **description is read at selection time**. It says what the tool does, when to use it, what it returns, and includes a **boundary clause** naming what it **doesn't** handle and which tool does. **Three or four sentences.**
- **Which tool** is decided by the description; **complete arguments** come from the schema. Two problems, two places to fix them.
- **`required`** means the operation **makes no sense without the field**. Too many required fields force made-up values; too few produce incomplete calls. **Enums, formats and `input_examples`** remove guesses. **`strict: true`** guarantees schema-valid arguments, within a JSON Schema subset.
- **Validate in the dispatcher**, your last point of control. Add **approval** for irreversible actions, plans about to run, and out-of-range results. A **denial with a reason** helps Claude; silence doesn't. Parse arguments; never compare raw JSON strings.
- Return **`is_error: true`** with a short, **actionable** message. Placeholder data, empty successes and silently removed tools all say something false. **Retry transient failures inside the tool**, with backoff and a limit.
- **`tool_choice`** has `auto`, `any`, `tool` and `none`, but **Opus 5.5, Sonnet 5.5 and Fable 5.1+ reject `any` and `tool`** with a 400. Enforce required steps **in your code**.
- A tool set **costs twice**: overlap hurts selection, and every definition is billed every request. Fifty tools can use **10–20K tokens**, and accuracy drops past **30–50**. **Consolidate** one resource's operations under an `action` parameter, never unrelated jobs into a catch-all. Use **tool search** above about **20 tools**; send everything below about **10**.

## Endnotes for this chapter

1. Tool use is a contract in which the model emits a structured request and the application or Anthropic's servers execute it; client-executed tools need an application loop keyed on `stop_reason`, while server-executed tools run their own loop, return `pause_turn` at the iteration limit, and their `server_tool_use` blocks need no `tool_result`. Anthropic, *How tool use works* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works)

2. A tool definition has a `name`, a detailed `description` and an `input_schema`; the API builds a system prompt from the definitions; guidance covers namespacing with common prefixes and grouping related operations under one tool with an `action` parameter; `input_examples` are validated against the schema, and invalid examples return a `400`; `tool_choice` accepts `auto`, `any`, `tool` and `none`. Anthropic, *Define tools* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)

3. A `tool_use` block carries an `id`, a `name` and an `input`; the reply is a user message of `tool_result` blocks, each with the `tool_use_id` it answers, optional `content`, and an optional `is_error` flag. Anthropic, *Handle tool calls* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls](https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls)

4. The wrong tool being called is attributed to ambiguous descriptions and fixed by differentiating tools by when to use them; out-of-schema parameters are fixed with `strict: true` or `input_examples`; a strict-tool `pattern` using a backreference, lookaround, word boundary or large range can't compile; every `tool_use` needs a `tool_result`, and results must come before any text in the user message; comparing serialised tool inputs as strings fails across model versions because escaping differs. Anthropic, *Troubleshooting tool use* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/troubleshooting-tool-use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/troubleshooting-tool-use)

5. `strict: true` on a tool definition constrains sampling to schema-valid output; without it Claude may return a string in place of an integer or omit required fields. Anthropic, *Strict tool use* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use)

6. Tool-use tokens come from the `tools` parameter, `tool_use` blocks and `tool_result` blocks, plus an automatically included tool-use system prompt whose size varies by model and `tool_choice`. Anthropic, *Tool use with Claude* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)

7. Tool search keeps definitions out of context until Claude asks for them, which suits toolsets of roughly twenty or more tools where most aren't needed every turn. Anthropic, *Manage tool context* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/manage-tool-context](https://platform.claude.com/docs/en/agents-and-tools/tool-use/manage-tool-context)

8. Fifty tools can use ten to twenty thousand tokens, and selection accuracy degrades with more than thirty to fifty loaded at once; tool search adds one round trip the first time Claude discovers a tool, and below about ten tools loading everything is typically faster. Anthropic, *Scale to many tools with tool search* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/tool-search](https://code.claude.com/docs/en/agent-sdk/tool-search)

9. A `canUseTool` callback pauses execution until it returns; it can allow the tool, allow it with modified input, or deny it with a message Claude sees and may use to adjust. Anthropic, *Handle approvals and user input* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/user-input](https://code.claude.com/docs/en/agent-sdk/user-input)

10. Guidance on describing a tool as you would to a new colleague, namespacing related tools with measurable effects on evaluations, returning specific, actionable error messages rather than opaque codes or tracebacks, and consolidating functionality into fewer high-impact tools. Anthropic, *Writing effective tools for AI agents* (Anthropic Engineering, accessed 2026-09-09; not re-checked for this edition), [https://www.anthropic.com/engineering/writing-tools-for-agents](https://www.anthropic.com/engineering/writing-tools-for-agents)

11. Claude Opus 5.5 doesn't support forced tool use: `tool_choice` set to `any` or a named `tool` returns a 400 `invalid_request_error`, including on the token counting endpoint, while `auto` and `none` are supported; the migration guide recommends `auto` with strict tool use or structured outputs, and saying in the prompt when the tool applies. The same restriction applies from Fable 5.1 onward; Opus 5 still accepts all four forms. With the `inline-tools-2026-09-15` beta header, a `tool_addition` block in a mid-conversation system message can carry a full tool definition, so tools can be added or changed mid-conversation without editing `tools` or losing the prompt cache. Anthropic, *What's new in Claude Opus 5.5* and *Migration guide* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/models/opus-5-5/whats-new-opus-5-5](https://platform.claude.com/docs/en/models/opus-5-5/whats-new-opus-5-5); OpenRouter, *Claude 5.5 migration* (accessed 2026-11-27), [https://openrouter.ai/docs/cookbook/evaluate-and-optimize/model-migrations/claude-5-5](https://openrouter.ai/docs/cookbook/evaluate-and-optimize/model-migrations/claude-5-5)
