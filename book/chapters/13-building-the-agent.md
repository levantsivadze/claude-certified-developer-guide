# Chapter 13: The Loop, the Harness, and What You Stop Owning

**Summary**: *There are three ways to run the same agent loop, and choosing between them is accounting, not taste. Adopt a framework and the loop, the tool dispatch and the history leave your codebase, while a subprocess model, a configuration format and someone else's data-handling policy arrive in their place. Get the accounting wrong and you rebuild half of what you adopted, or send regulated data through a runtime nothing covered. This chapter reads both sides of that ledger, then puts hooks where rules have to hold.*

## Adopting the SDK Deletes Code and Adds Assumptions

The agent loop is about **sixty lines**: send the messages, read the reply, run any requested tools, append the results, send again. It's the first thing you write against a tool-using model, and it works by the second afternoon.

Then a teammate suggests the **Agent SDK**, and those sixty lines leave the repository. So does the **dispatch table** mapping tool names to functions, and the list that **accumulated conversation history**. Three things nobody maintains any more.

None of them stopped existing. The SDK runs the **same loop**, in the same process, against the same model.<sup>[1]</sup> What changed is **who fixes it** when it breaks. And something moved the other way: a set of **assumptions your application now carries**, such as a session mapping to a running process, transcripts landing on local disk, and tools being declared rather than dispatched.

This book calls this the **ownership ledger**. Every construction decision has **two columns**:

- **What leaves:** the loop, the sandbox, the retries, the runtime.
- **What arrives instead:** an execution model, a configuration format, a data-handling policy, and someone else's release schedule.

A choice made by reading only the first column isn't a choice. **Both columns are always full.**

## The Loop Ends Only When a Reply Has No Tool Calls

Learn the loop well enough to recognise it whoever wrote it, because every decision in this chapter is about which parts someone else wrote.

Claude receives the prompt, plus the system prompt, tool definitions and any history. It replies with text, tool requests, or both. The requested tools run and their results go back. The cycle repeats, and finishes when a reply contains **no tool calls**.<sup>[2]</sup> One full cycle is a **turn**.<sup>[2]</sup>

```
repeat:
    send     system prompt + tool definitions + history + newest input
    receive  the reply
    stop     if the reply requests no tools
    run      every tool the reply requested
    return   one result per request, all together, before the next reply
    check    turn count and spend against their limits
```

**Four things must be wired up** whoever runs the loop, and the fourth is the one people skip once the first three work:

1. **Tools registered**, each with a schema for its arguments.
2. A **system prompt scoped** to the task and its tools. A broad one produces broader, less reliable tool choice.
3. **Every tool call run and answered**, and all calls from one reply resolved together before the next reply.
4. **Exit conditions**, because a loop with no stopping rule keeps asking for tools after the work is done.

At the end, your application branches on the **result's subtype**: `success`, `error_max_turns`, `error_max_budget_usd`, or `error_during_execution`. The **final text only exists on success**.<sup>[2]</sup> Code that reads the text without checking the subtype works perfectly until the first day a limit is hit. **Cost and usage come with every subtype**, including failures, so a run that ended badly is still billed.<sup>[2]</sup>

> **Example.**
> ```python
> async for message in query(prompt=task, options=options):
>     if message.type == "result":
>         if message.subtype == "success":
>             save_report(message.result)
>         else:
>             alert(f"Agent stopped early: {message.subtype}, cost ${message.total_cost_usd:.2f}")
> ```

## One SDK Session Is One Subprocess, With a Shell, a Folder and a Transcript

The SDK isn't a thin wrapper around an HTTP call, and treating it like one is where production surprises come from. Starting a query **launches a separate CLI process** that the SDK talks to over standard input and output. That process owns a **shell**, a **working directory** and **session transcripts on local disk**.<sup>[3]</sup> So how many sessions a host can run is limited by memory, and each keeps its own filesystem state.<sup>[3]</sup>

Nobody asked for a subprocess. It came with the loop, and it decides every hosting question later in this chapter. That's the ledger's second column in practice.

Three more things arrive with it:

- **Tools become declarations.** A custom tool has four parts: a name, a description, an argument schema and the function that runs.<sup>[4]</sup> If that function **throws**, the loop doesn't crash: the SDK catches it and gives Claude an **error result** to react to.<sup>[4]</sup>
- **Availability and permission become separate settings.** Leaving a tool out of the list keeps it **out of context**, so Claude never tries it. A **permission rule** that blocks calls leaves the tool **visible**, so Claude can waste a turn trying something that will never run.<sup>[4]</sup> Chapter 14 covers permission modes.
- **Continuity becomes an ID.** **Continue** picks up the most recent session in the working directory. **Resume** returns to a specific session ID. **Fork** copies the history to a new ID and diverges, leaving the original intact.<sup>[5]</sup> All three keep the **conversation**; none of them keeps the **filesystem**. A forked run that edits a file has edited it for every run in that folder.<sup>[5]</sup>

Cost tracking changes too. Each query reports its own total, and **there's no session-level total**, so you add them up yourself. Once subagents are involved, the plain `usage` figure only covers the main loop; use the **per-model breakdown** for the whole tree.<sup>[6]</sup>

## Only Build Your Own Loop for a Constraint the Library Can't Meet

Writing the loop yourself against the **Messages API** gives you full control, and full responsibility. Context management, parallel tool handling, retries and exit conditions all become code your team writes, tests and maintains. Your own loop is worth it when a **constraint rules out the library**: a deployment that can't accommodate what the SDK assumes, or a need to understand the loop before any abstraction sits on top.

The expensive mistake usually isn't picking the wrong path. It's picking a path and then **rebuilding the parts that were the reason for choosing it**:

- **The framework used only for its types.** A team adopts the SDK, then hand-writes the loop and history anyway. Now there are **two sources of truth** for the conversation, and no reviewer can say which one Claude actually saw.
- **Structured calls swapped for prose.** Asking Claude to describe its intention in text and then parsing that text throws away the **schema**, the one thing that made the request machine-readable.
- **A storage layer nobody asked for.** Moving history into a separate store, with no durability requirement, splits ownership of the state. Keeping the two copies in sync becomes a feature with a maintainer.

Each looks like diligence, and each is more code than either honest option. Once you've chosen a path, **let it own the loop, the dispatch and the history together**.

## Data Rules Can Rule Out a Deployment Before Preference Even Matters

**Claude Managed Agents** is the third path, and it flips the arrangement. Anthropic runs the **loop** and the **execution sandbox**. You define the agent once as a **versioned resource**, refer to it by ID, and your application **sends events in and streams results back**.<sup>[7]</sup> It has four parts:<sup>[7]</sup>

- **Agent**: model, system prompt, tools, MCP servers and skills.
- **Environment**: where sessions run.
- **Session**: one running instance doing one task.
- **Events**: messages in both directions.

Now the fact that decides scenarios. Managed Agents sessions are **stateful by design**: conversation history, sandbox state and outputs are **stored on Anthropic's side**, and transcripts persist **until you delete them**. That's why Managed Agents (including self-hosted sandboxes) is **not eligible for Zero Data Retention or HIPAA readiness**.<sup>[7]</sup><sup>[12]</sup> If the scenario involves **protected health information** or a zero-retention commitment, this path is **ruled out by a fact**, not by judgement. Nothing about how well it fits operationally changes that.

**Self-hosted sandboxes** sit in between. **Orchestration stays with Anthropic**, while **tool execution moves onto infrastructure you control**, so the filesystem, the processes and the network the agent can reach fall under your policy.<sup>[8]</sup> The part people misread: **Claude still sees every tool input and result**, because it can't choose the next step without them.<sup>[8]</sup> Moving execution inside your boundary isn't the same as moving the conversation inside it.

| The real question | Managed, cloud sandbox | Managed, self-hosted sandbox | SDK, self-hosted |
|---|---|---|---|
| Who runs the loop | Anthropic | Anthropic | Your application |
| Where tool code runs | An Anthropic sandbox | Your host | Your host |
| What limits network access | Anthropic's egress controls | Your firewall | Your firewall |
| Who hardens the image and runtime | Anthropic | You | You |
| Whose on-call gets paged | Anthropic's | Yours, for the worker | Yours |

Self-hosting is **shared responsibility**, not a handover. Anthropic **doesn't inspect your image**. **Egress limits are yours**; without them, a compromised tool can reach any host. The **environment key** that lets your worker claim work and post results belongs in a **secrets manager**. Conversation content and tool output pass through your worker, so **retention, redaction and deletion** of that data are your job.<sup>[9]</sup> Anthropic states four things it **can't** do for you: know your key leaked, verify what's in your image, isolate one tool execution from another inside your sandbox, or enforce retention on data that has reached your worker.<sup>[9]</sup>

Self-hosting the **SDK** teaches the same lesson on a smaller scale. **Local disk doesn't survive** a restart, a scale-down or a move to another node, and a session store **only mirrors transcripts**, not memory files or other working-directory files.<sup>[3]</sup> In a container serving **several tenants**, filesystem settings and project files can carry one tenant's data into another's session, unless you **turn those settings off** and give each tenant its **own working directory and configuration directory**.<sup>[3]</sup>

Two named failures cover most wrong answers here:

- **Compliance deferred to a later migration**: "pilot on the convenient path, move later". That processes regulated data outside policy for the whole pilot.
- **The exception assumed granted**: drafting a policy exception while the deployment goes ahead. Same violation, with paperwork.

## A Hook Fires at a Fixed Point, Whether or Not the Model Cooperates

A **hook** is a callback the runtime calls at a **named moment** in the loop: before a tool runs, after it returns, when a prompt is submitted, when the agent stops, when a subagent starts or stops, and before compaction.<sup>[2]</sup> Hooks run in **your application's process**, not in Claude's context window, so they **cost no tokens**.<sup>[2]</sup>

A hook **isn't a request**, which makes it the answer to a whole class of scenario. A callback that runs **before a tool executes** can **refuse it**, and the tool **doesn't run**. Claude gets the refusal as the tool's result and usually tries something else.<sup>[10]</sup> An instruction in the system prompt is **guidance** Claude weighs against everything else. A hook is **code that executes**.

> **Example.** A `PreToolUse` hook that blocks any shell command touching production:
> ```python
> async def block_prod(input_data, tool_use_id, context):
>     cmd = input_data["tool_input"].get("command", "")
>     if "prod" in cmd:
>         return {"hookSpecificOutput": {
>             "hookEventName": "PreToolUse",
>             "permissionDecision": "deny",
>             "permissionDecisionReason": "Commands against production are not allowed."}}
>     return {}
> ```
> Registered with a matcher for the `Bash` tool, this runs on every shell command, however the prompt was worded.

Four details decide the rest:

- **Position decides power.** Only a check **before** the action can stop it. A callback **after** it can record what happened, replace the result before Claude reads it, or add context, but it **can't undo** what already happened. **Audit offered as prevention** is the mistake: the logging is real, but it's one step late.
- **The most restrictive answer wins.** When several hooks and rules apply, **deny beats defer, defer beats ask, ask beats allow**, and a single deny blocks the call.<sup>[10]</sup>
- **Order isn't guaranteed.** Matching hooks run **in parallel** and finish in **no fixed order**, so each must work without assuming another ran first.<sup>[10]</sup>
- **Matchers filter on tool names, not arguments.** Anything about a path, command or value is checked **inside the callback**.<sup>[10]</sup> A hook that runs **asynchronously** (without blocking the loop) can't refuse or change anything, because the loop has moved on; it's only for **logging, metrics and notifications**.<sup>[10]</sup>

Managed Agents reaches the same goal with a **permission policy** on each toolset: `always_allow`, or `always_ask`, which **pauses the session** until someone approves. The built-in agent toolset **allows by default**; **MCP toolsets ask by default**, so a tool added to a server later **doesn't run unreviewed**.<sup>[11]</sup> **Custom tools** your application runs itself sit **outside permission policies**, so that approval logic is back in your code.<sup>[11]</sup> Chapter 21 returns to hooks as a safety mechanism. For now, one rule: **name the moment**, then check whether it falls **before or after** the thing you were asked to control.

## How the Exam Builds Wrong Answers

Wrong answers here describe real engineering; several answer a question one clause away from the one asked. Four assumptions produce almost all of them:

- **Preference weighed against a constraint that already decided.** The scenario names a residency rule, a retention commitment or a regulator, and an option treats it as one factor to balance against speed. A requirement like that **sets the allowed options** before anything is compared.
- **Half-adoption dressed as caution.** A path is chosen, then one of its core parts is rebuilt by hand or supplemented with a store nobody asked for. Result: **two owners for one piece of state**.
- **Recording mistaken for preventing.** Logs, traces and after-the-fact review, offered where the scenario said something must be **stopped**. A refusal has to arrive while the call is still a request. Read the **verb** in the requirement.
- **A data boundary inferred from where code runs.** Assuming that running tools locally means **nothing reaches the provider**, or that a managed runtime inherits coverage the account already has. Ask what each component **must see** to do its job; that's the boundary.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| A residency, retention or regulatory rule applies, and a hosted option is offered | Use the deployment the rule allows, then reduce its operational cost | Balancing the rule against speed, piloting first, or assuming an exception |
| The team has adopted a framework and is now wiring tools | Let it own the loop, dispatch and history | Hand-writing any of the three, or adding an unrequested store |
| A specific action must never happen | A check before the tool runs, able to refuse it | Logging it afterwards, or forbidding it in the system prompt |
| Execution must stay inside a network boundary; orchestration may be outside | Move tool execution onto infrastructure you control | Assuming nothing then reaches the provider |
| A long task shouldn't hold a local process open, and no data rule applies | A managed runtime that owns the session and sandbox | Ruling it out by preference, or choosing it with regulated data |
| The run must stop on a stated condition | An explicit limit the runtime enforces and reports | Expecting Claude to stop when the work looks done |
| Cost must be attributed across a run with subagents | Whole-tree accounting across every model used | The main loop's `usage` figure, which misses nested work |
| A rule must apply to tools nobody has added yet | A default that asks rather than allows | An allow list naming today's tools |

One habit beats the table. **Name the constraint** the scenario states before comparing anything, then ask which options survive it. Most wrong answers here are sound engineering that the constraint had already ruled out.

## What to Remember

- The **ownership ledger**: every path has two columns, **what leaves your code** and **what arrives as an assumption**. Adopting the SDK removes the **loop, dispatch and history**, and adds a **subprocess per session** with **transcripts on local disk**.
- The **agent loop**: inputs go in, Claude answers or requests tools, results go back, and it **ends only when a reply has no tool calls**. One cycle is a **turn**.
- **Four things to wire** on every path: **tools with schemas**, a **scoped system prompt**, **every tool call answered together**, and **explicit exit conditions**.
- **Branch on the result subtype**, not the text: `success`, `error_max_turns`, `error_max_budget_usd`, `error_during_execution`. Text only on success; **cost on all four**.
- **Build your own loop** only for a **constraint the library can't meet**. It costs you context management, parallel tools, retries and exits as maintained code.
- **Half-adoption is the expensive answer**: rebuilding the loop or history after choosing a framework creates **two sources of truth**; parsing intent from prose **throws away the schema**.
- **Managed Agents** runs the loop and sandbox, with agents as **versioned resources**. Sessions are **stored server-side until you delete them**, so Managed Agents (self-hosted sandboxes included) **isn't eligible for ZDR or HIPAA readiness**. That rules it out for regulated data, whatever else fits.
- **Self-hosted sandboxes** move **tool execution** to your infrastructure, but **Claude still sees every tool input and result**. You own **image hardening, egress, key storage and retention**.
- **Hooks** run at named points in **your process**, cost **no tokens**, and the **most restrictive decision wins**. They run **in parallel**, in no set order, and **matchers match tool names**, not arguments.
- **Only a check before the action can refuse it.** Logs and traces answer a **reporting** requirement, never a **prohibition**.

## Endnotes for this chapter

1. The Agent SDK makes the agent loop and context management behind Claude Code available as a library in Python and TypeScript. Anthropic, *Agent SDK overview* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/overview](https://code.claude.com/docs/en/agent-sdk/overview)

2. The loop repeats until a response has no tool calls; a turn is one round trip that completes without yielding to the caller; result subtypes are `success`, `error_max_turns`, `error_max_budget_usd` and `error_during_execution`, with result text only on `success` and cost, usage, turn count and session ID on every subtype; hooks run in the application process, not the context window, and consume no context. Anthropic, *How the agent loop works* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/agent-loop](https://code.claude.com/docs/en/agent-sdk/agent-loop)

3. The SDK spawns a `claude` CLI subprocess over stdio that owns a shell, a working directory and JSONL transcripts on local disk, one per session, with concurrency bounded by memory; local disk doesn't survive restarts, scale-down or node moves; a `SessionStore` mirrors transcripts only; multi-tenant isolation needs filesystem settings and auto memory disabled and per-tenant configuration and working directories. Anthropic, *Hosting the Agent SDK* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/hosting](https://code.claude.com/docs/en/agent-sdk/hosting)

4. A custom tool has a name, description, input schema and async handler on an in-process MCP server; uncaught handler exceptions become error results rather than stopping the loop; availability (whether a tool appears in context) is separate from permission (whether a call is approved), and a block rule leaves the tool visible. Anthropic, *Give Claude custom tools* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/custom-tools](https://code.claude.com/docs/en/agent-sdk/custom-tools)

5. Continue finds the most recent session in the directory, resume takes a session ID, and fork copies history to a new session; sessions persist the conversation, not the filesystem, so a forked agent's file edits are visible to other sessions in the same directory. Anthropic, *Work with sessions* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/sessions](https://code.claude.com/docs/en/agent-sdk/sessions)

6. Each `query()` returns its own `total_cost_usd`, with no session-level total; `usage` covers only the main loop, and `modelUsage`/`model_usage` is needed for whole-tree accounting. Anthropic, *Track cost and usage* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/cost-tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking)

7. Managed Agents is a pre-built agent harness on managed infrastructure for long-running and asynchronous work, using the `managed-agents-2026-04-01` beta header; its concepts are agent, environment, session and events; it is stateful by design, storing conversation history, sandbox state and outputs server-side, and for that reason is not currently eligible for Zero Data Retention or HIPAA coverage. Anthropic, *Claude Managed Agents overview* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/managed-agents/overview](https://platform.claude.com/docs/en/managed-agents/overview)

8. Self-hosted sandboxes keep orchestration on Anthropic's side while tool execution runs on customer infrastructure; tool inputs and outputs still flow to Anthropic's control plane so the model can decide the next step. Anthropic, *Self-hosted sandboxes* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes)

9. In the shared responsibility model, Anthropic secures the control plane while image quality, runtime hardening, egress controls, environment key storage and rotation, workload isolation, credential handling and log retention fall to the customer; Anthropic can't know a key leaked, verify the worker build, isolate tool executions inside the customer's sandbox, or enforce retention once data reaches the worker. Anthropic, *Security model* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes-security](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes-security)

10. A `PreToolUse` deny stops the tool call and passes a reason to the model; deny beats defer, defer beats ask, ask beats allow, and any deny blocks the call; matching hooks run in parallel in no fixed order; matchers match tool names, not arguments; async hooks can't block or modify because the agent has moved on. Anthropic, *Intercept and control agent behavior with hooks* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/hooks](https://code.claude.com/docs/en/agent-sdk/hooks)

11. Permission policies `always_allow` and `always_ask` control server-executed tools; the agent toolset defaults to `always_allow` and MCP toolsets to `always_ask`, so newly added MCP tools don't run without approval; custom tools run by the application aren't governed by permission policies. Anthropic, *Permission policies* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/permission-policies](https://platform.claude.com/docs/en/managed-agents/permission-policies)

12. Zero Data Retention does not cover Claude Managed Agents, a stateful resource whose session transcripts persist until deleted, with no automatic deletion; the feature eligibility table lists Managed Agents (`/v1/agents`, `/v1/sessions`, `/v1/environments`) as eligible for neither ZDR nor HIPAA readiness, across all sub-features including self-hosted sandboxes. Anthropic, *API and data retention* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/manage-claude/api-and-data-retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention)
