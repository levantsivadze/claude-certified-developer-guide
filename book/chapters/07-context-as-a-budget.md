# Chapter 7: The Context Window Is a Budget, and Everything Competes for It

**Summary**: *By the twentieth tool call, most of an agent's context window holds work that's already finished. It was useful once. You still pay for it on every request, and it still competes for attention with the rule you set three turns ago. This chapter covers where the budget goes, why overspending shows up as odd behaviour instead of an error, the two tools that free up room, and what compaction is allowed to throw away.*

## Everything in a Request Uses the Same Budget, Including Finished Work

Take an agent twenty tool calls into a task. Before you even add the current instruction, the window holds:<sup>[1]</sup>

- the **system prompt**
- every message, including **tool results, images and documents**
- your **tool definitions**
- the turn's **output**, including **thinking**

**Cached prefixes** count too. Caching changes what those tokens cost, not whether they take up room.<sup>[1]</sup>

Most of that has already done its job. This book calls it **spent context**: content fetched for a step that's finished, **resent on every later request**, using the same budget as the question you actually care about. Anthropic's guidance on context engineering frames this as an **attention** problem, not a storage problem. Every token draws on a limited attention budget, so the work is keeping the **smallest set of useful tokens**, not finding more room.<sup>[4]</sup>

Development hides this for one reason: **test data is small**. Real tool outputs are often several times longer than test fixtures. So a session that held together for twenty turns in testing **fills up by turn eight** in production. The window didn't change. Each turn just uses more of it.

That's why you measure instead of guessing. The **token counting endpoint** takes the same body as a real request and returns a count without running the model.<sup>[6]</sup>

> **Example.** A made-up support agent reads order records. In testing, each record is about 800 tokens. In production, records include full shipping histories and average 6,000 tokens. Twenty lookups were 16,000 tokens in testing and 120,000 in production. Same code, same window, very different behaviour.

## Bloat Is a Size Problem; Drift Is an Attention Problem

Overspending shows up in two ways, and only one of them raises an alarm.

**Bloat** is size. Requests grow, the bill grows, latency grows. If the input alone ever **exceeds the window**, the API returns a **`400`**.<sup>[1]</sup> All of this appears on a dashboard.

**Drift** is behaviour, and it **reports nothing**. Claude answers from details settled ten turns ago, quietly **stops following a rule** set at the start, or **picks the wrong tool** for a step it handled fine earlier. Anyone reading the transcript sees work that looks reasonable.

| The symptom | What it means | The wrong conclusion it invites |
|---|---|---|
| Each turn costs more than the last on the same task | Finished material is being resent in full | Blaming the model tier for latency |
| Tool choices are right for four turns, then wrong on the fifth | Old results now outweigh the instructions | Rewriting the tool descriptions |
| Answers lean on old details nobody has mentioned in ten turns | Old details still carry full weight | Calling it a retrieval or memory bug |

The middle row eats the most debugging time. The tool schema didn't change between turn four and turn five. What changed is **how much finished work sits between the instructions and the decision**.

That's why **a bigger window doesn't fix drift**. A window twice the size keeps every distraction at full detail and leaves room for more. Chapter 1 separated capacity from attention for a single request; this is the same idea across a whole run.

The same reasoning answers a common proposal: putting a corpus of tens of thousands of documents into every prompt. Even if it fits, you **pay input on the whole corpus** for every query and spread attention across all of it. Retrieve the relevant passages instead.

## Two Tools Free Up Room: Delete Finished Material, or Summarise It

**Context editing deletes. Compaction rewrites.** Exam questions often turn on that difference.

**Clearing tool results** removes the oldest tool calls and results once the prompt passes a trigger (**100,000 input tokens** by default), keeping the **three most recent** by default.<sup>[2]</sup> Each cleared result leaves **placeholder text**, so Claude knows something was removed rather than thinking it never happened.<sup>[2]</sup> The parameters are where your judgement goes:<sup>[2]</sup>

- **`exclude_tools`** lists tools whose results are **never cleared**.
- **`clear_at_least`** skips clearing unless it would free a worthwhile number of tokens.
- **`clear_tool_inputs`** clears the call parameters as well as the results.
- A companion strategy, **`clear_thinking_20251015`**, clears old thinking blocks.

All of this runs **on the server**. Your application keeps its own full history and never has to resync.<sup>[2]</sup>

**Compaction summarises** instead. It triggers on input tokens (**150,000** by default, **minimum 50,000**). Claude writes a summary into a **`compaction` block** that you send back on later requests, and **everything before that block is ignored**.<sup>[3]</sup> It costs an **extra model call**, billed and counted against rate limits, and it always uses the model in your request; you can't pick a cheaper one.<sup>[3]</sup> It's still a beta feature (`compact-2026-01-12`). With **`pause_after_compaction`**, the API stops after writing the summary, so you can add content, such as the latest messages or a key instruction, before it continues.<sup>[3]</sup>

| | Clearing tool results | Compaction |
|---|---|---|
| What happens to old material | Removed, with a placeholder marking the gap | Rewritten as a summary; earlier blocks are ignored |
| What survives | Whatever your parameters protect | Whatever the summary happened to include |
| What it costs | A cache miss from the point where content was cleared | An extra billed model call |
| Use it when | Old tool output is most of the growth | The conversation is long and continuity still matters |

> **Example.** A coding agent reads 40 files while hunting a bug. Once it's found the bug, the file contents are spent. Clearing tool results drops them and leaves placeholders, but keeps the last three reads. Compaction would instead write "searched the auth module; bug is in `token.py` line 88", and keep the conclusion without the files.

**Tool definitions** are a third source of pressure, and neither tool touches them, because they sit at the **front of every request**. The documented fixes are **loading tools only when Claude asks for them** (tool search) or **collapsing a chain of calls into one sandboxed script** (programmatic tool calling).<sup>[5]</sup> Chapter 9 covers those; chapter 8 covers caching.

## A Rule That Only Lives in the Conversation Can Be Summarised Away

**Compaction keeps the gist.** That's the point of it, and also the risk.

Run a long task through several compactions. A rule stated once, in turn three, gets summarised, and then **summarised again from that summary**. Each pass is free to decide the rule wasn't worth keeping. **Nothing raises an error** when it disappears. The first sign is an agent calmly breaking a rule it followed an hour ago.

The tempting fix is to **repeat the rule in the conversation**, perhaps with "remember this". But the repeated rule sits **inside the same region being summarised**, so it's exposed to the same process that dropped it.

So the fix is about **where the rule lives, not how loudly it's stated**. A rule that must hold for the **whole run** belongs **outside the part compaction rewrites**: in the **system prompt**, or in a record your application re-sends every turn.

> **Example.** "Never modify files under `migrations/`" is said once in turn 3. After two compactions, the summary reads "working on database schema changes". The rule is gone, and the agent edits a migration. Putting the rule in the system prompt keeps it safe, because compaction never rewrites the system prompt.

**Changing the trigger threshold** looks like another fix, but it isn't. A higher trigger compacts less often, on more material. A lower one compacts more often, on less. Both change **how often** the loss happens. Neither changes **what a summary may leave out**.

**Custom summary instructions replace the default prompt entirely**; they don't add to it.<sup>[3]</sup> If you just write "summarise the conversation", you get a readable story that drops the **file paths, decisions and errors already fixed**. Ask for those explicitly. On Claude 5.1 and later, a summary written with custom instructions only sees the visible conversation, not earlier thinking.<sup>[3]</sup>

Claude Code shows the same boundary clearly. **Path-scoped rules and nested `CLAUDE.md` files** load into the conversation when their trigger file is read, so **compaction can summarise them away**. A rule that must survive belongs in the **project-root `CLAUDE.md`**. Skill bodies are re-added after compaction, but a large one is **cut to a size cap, keeping the start**, so put the important instructions at the top.<sup>[8]</sup>

## Isolation Keeps Material Out of the Window Instead of Shrinking It

Clearing and compaction both work on material that's already arrived. The third approach **stops it arriving**.

A **subagent** runs in its **own conversation**. Its tool calls and results stay there, and **only its final message comes back** to the parent. So a subagent can read dozens of files without any of that content landing in the caller's window.<sup>[7]</sup> You get the answer without the journey.

**Separate agents share no context.** The only thing that crosses from parent to subagent is the **prompt string** that launched it. File paths, error text and decisions already made are only there if you put them there.<sup>[7]</sup> The mistake is the **assumed handoff**: designing as if a later step can see what an earlier one found, just because both ran in the same job.

> **Example.** The parent agent finds that tests fail with `ImportError: no module named 'pydantic'`. It launches a subagent with "fix the failing tests". The subagent starts fresh, reruns everything to rediscover the error, or guesses. A better prompt is: "Tests fail with `ImportError: no module named 'pydantic'` in `tests/test_api.py`. Add the dependency to `pyproject.toml` and rerun only that file."

**Multi-step workflows** get the same benefit without separate agents, by giving each step **only what its own job needs**. When reviewing a long document, narrow passes over each section catch local errors, while a **contradiction between two sections** only shows up in a pass whose job is that comparison. Running the whole document three times and voting just repeats one attention problem three times. Agreement on something everyone missed isn't detection.

Chapter 12 covers choosing between a workflow and an agent; chapter 13 covers building the loop. The cost of isolation is **visibility**: the subagent's reasoning disappears with its context.

## Prompt Engineering Shapes One Request; Context Engineering Shapes the Whole Run

These two terms get used as synonyms, but they describe **different jobs at different scales**.

- **Prompt engineering** writes **one exchange**: the wording of an instruction, the examples beside it, the shape of the answer. Its unit is **one request**, and its question is **what to say**. Anthropic's own guidance notes that not every failing result is best fixed by changing a prompt.<sup>[9]</sup>
- **Context engineering** decides **what's in the window across a run**: what gets fetched, what survives the next turn, what's compacted, what's handed to a subagent so it never lands here. Its unit is the **whole session**, and its question is **what should be present**.<sup>[4]</sup>

On a single call the two blur together. They separate once a task runs long. A prompt can be perfect and the run can still degrade, because by turn forty the instruction is competing with thirty-nine turns of finished work. No rewrite of the instruction reaches that. **Length decides which lever matters**: for short single-turn tasks, the lever is the wording; for long multi-step tasks, it's what you let pile up.

## How the Exam Builds Wrong Answers

Wrong answers here are competent engineering aimed at the wrong thing. Four assumptions produce almost all of them:

- **More capacity offered when the problem is competition.** A larger window, a bigger model, extended context. Each helps when material doesn't fit, and none helps when it fits but **drowns out what matters now**.
- **A frequency dial offered against a loss.** Raising the compaction threshold, lowering it, compacting on a schedule. All three change **how often** summarising runs, not **what it may drop**.
- **Throwing away continuity to cut volume.** Restarting every few calls, keeping only the latest turn, dropping tool calls. Growth stops because the **task state went with it**.
- **Assuming context travels.** Treating a later step, a separate agent or a post-compaction turn as able to see what an earlier one found. Nothing carries over unless **something carries it**.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| An agent degrades after a fixed number of tool calls, and the window is already big enough | Clear spent tool output or compact, protecting the current task state | A larger window, a different framework, merged tools |
| A rule is broken only after several compactions | Move it outside what compaction rewrites: the system prompt or a re-sent record | A higher threshold, or telling the model to remember it |
| History is resent in full and requests keep growing | Summarise or trim older turns, keeping what's still in use | Capping reply length, or discarding history entirely |
| A step must work without seeing the whole earlier conversation | Give each step only what it needs, via a subagent or a multi-step workflow | One global prompt handed to every step |
| A corpus far bigger than any window is proposed for every prompt | Retrieve the relevant passages per query | A document-count limit, or splitting the corpus across parallel requests |
| A later step depends on something an earlier step found | Pass it explicitly in the prompt that starts the step | Assuming it's visible because it happened in the same run |
| Answers keep leaning on old details from many turns ago | Compact, so the gist stays and the details lose weight | Keeping only the latest turn, or resetting every turn |
| A prompt is already precise, and a long multi-step run still drifts | Change what the window holds, not what the instruction says | Rewriting or lengthening the instruction |

One habit works beyond the table. For any option, ask whether it changes **how much room there is**, **how often** something is discarded, or **what** may be discarded. Drift only responds to the third.

## What to Remember

- The window is **one budget** shared by the **system prompt**, all messages (including **tool results, images and documents**), **tool definitions**, and the turn's **output and thinking**. **Cached prefixes still take up room**; caching changes the price, not the space.
- **Spent context** is material that has **already done its job** but is **still resent every turn**, still costing tokens and competing for attention. It's most of a long agent run. Measure it with the **token counting endpoint**.
- **Bloat is size; drift is behaviour.** Bloat raises cost and eventually a **`400`**. Drift **reports nothing** and shows up as old details, dropped rules and worse tool choices. **Tool choice failing after a fixed number of turns** is a window problem, not a schema problem.
- **A bigger window doesn't fix drift**, because it keeps every distraction at full detail.
- **Clearing** removes old tool results and leaves a **placeholder** (tuned with `keep`, `clear_at_least`, `exclude_tools`, `clear_tool_inputs`). **Compaction** rewrites history into a summary, **ignores every block before it**, and costs an **extra billed model call**.
- Compaction is **lossy by design**. Anything that must hold for the **whole run** goes in the **system prompt** or a **record re-sent each turn**. Repeating the rule in the conversation puts it back in the **region being summarised**, and threshold changes only change **how often** the loss happens. Custom summary instructions **replace** the default prompt.
- **Isolation keeps material out**: a subagent's tool calls stay in **its own context**, and only its **final message** returns. **Separate agents share no context**, so the **launch prompt is the only channel**. The cost is **visibility**.
- **Prompt engineering** shapes **one request** (what to say). **Context engineering** shapes **the whole run** (what should be present). On long runs, only the second reaches the problem.

## Endnotes for this chapter

1. Everything in the request counts toward the context window: the system prompt, every message including tool results, images and documents, and the tool definitions; the output for the turn, including extended thinking, counts too. Cached prompt prefixes still occupy the window, because caching changes what you pay rather than whether the tokens count. If the input alone exceeds the window, the API returns a `400` `invalid_request_error`. Anthropic, *Context windows* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/context-windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)

2. The `clear_tool_uses_20250919` strategy clears the oldest tool results once context passes a threshold, replacing each with placeholder text; `trigger` defaults to 100,000 input tokens and `keep` to 3 tool uses; `clear_at_least`, `exclude_tools` and `clear_tool_inputs` tune it; `clear_thinking_20251015` manages thinking blocks; editing runs server-side and the client keeps the full history; clearing invalidates the cached prefix from the point of clearing. Anthropic, *Context editing* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/context-editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)

3. Compaction is enabled with the `compact_20260112` strategy in `context_management.edits` (beta `compact-2026-01-12`); the trigger defaults to 150,000 input tokens and must be at least 50,000; the API returns a `compaction` block which must be passed back, and all content blocks before it are ignored; `pause_after_compaction` pauses after the summary so you can add content before the response continues; custom `instructions` completely replace the default summarisation prompt; on Claude 5.1 and later, requests with custom instructions summarise only the visible conversation, not earlier thinking blocks; compaction adds a sampling step that counts toward billing and rate limits, using the model in the request. Anthropic, *Compaction* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)

4. Context engineering is the set of strategies for curating the optimal set of tokens during inference; context is a finite resource with diminishing returns, drawing on a limited attention budget, and the aim is the smallest set of high-signal tokens. Compaction, structured note-taking and sub-agent architectures are given as long-horizon techniques. Anthropic, *Effective context engineering for AI agents* (Anthropic Engineering, accessed 2026-09-09; not re-checked for this edition), [https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

5. Tool definitions and accumulated tool results use context; tool search keeps definitions out until Claude asks for them, programmatic tool calling runs a sequence of calls as one sandboxed script so intermediate results never enter history, prompt caching reduces the cost of tokens already present, and context editing removes old tool results. Anthropic, *Manage tool context* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/manage-tool-context](https://platform.claude.com/docs/en/agents-and-tools/tool-use/manage-tool-context)

6. The token counting endpoint accepts the same inputs as message creation and returns the input token count without running inference; it is free, subject to rate limits. Anthropic, *Token counting* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/token-counting](https://platform.claude.com/docs/en/build-with-claude/token-counting)

7. Each subagent runs in its own conversation; intermediate tool calls and results stay inside it and only its final message returns to the parent. A non-fork subagent starts fresh, and the only content passed from parent to subagent is the Agent tool's prompt string, so file paths, error messages and decisions must be included in that prompt. Anthropic, *Subagents in the SDK* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/subagents](https://code.claude.com/docs/en/agent-sdk/subagents)

8. In Claude Code, path-scoped rules and nested CLAUDE.md files load into message history when their trigger file is read, so compaction can summarise them away; a rule that must persist belongs in the project-root CLAUDE.md. Skill bodies are re-injected after compaction but large skills are truncated to a per-skill cap, keeping the start of the file. Anthropic, *Explore the context window* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/context-window](https://code.claude.com/docs/en/context-window)

9. Prompt engineering guidance is scoped to success criteria controllable through prompting, and notes that not every failing evaluation is best solved by prompt engineering. Anthropic, *Prompt engineering overview* (Claude Platform Docs, accessed 2026-09-13; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview)
