# Chapter 12: Known Steps Need a Chain; Unknown Steps Need an Agent

**Summary**: *Two systems can use the same model, the same tools and the same goal, and still differ completely in who decides what happens next. Lean too far towards autonomy and you pay a model call to reach a conclusion someone reached years ago, on a bill that can run to fifteen times a chat once the work fans out. Lean too far the other way and the design breaks on the first input that doesn't fit its fixed route. This chapter is built around one question: what is genuinely unknown before the run starts?*

## Every Step You Can Write Down in Advance Is a Step No Model Needs to Choose

A team wants an assistant for **code review**. The sequence is already settled, because people have run it by hand for two years:

1. Pull the diff.
2. Check it against the style rules.
3. Look for tests that should have changed and didn't.
4. Summarise what changed.
5. Post the summary.

Five steps, same order, every time. **Writing them down settles the architecture.** When the sequence is known before the input arrives, a **chain of prompts** is **cheaper, repeatable and easy to inspect**. Each stage takes one input and produces one output. Run it twice and the request takes the same route both times. If something breaks, the broken stage has a name and a test case you can rerun.

Give those five steps to an agent instead, and the first thing it does is **spend a model call working out which step comes first**. That call reaches a conclusion the team reached two years ago. You pay tokens and latency for it, and you accept that Tuesday's run may decide differently.

This book calls it the **enumeration test**: **can you write the sequence down before the input arrives?**

- **Yes:** your **code holds the plan**, and the model does the work inside each step.
- **No:** the **model holds the plan**, and your **code holds the limits**.

Anthropic draws the same line by asking where control flow lives. A **workflow** runs models and tools along **paths fixed in code**; an **agent** directs its own process and decides how the task gets done.<sup>[1]</sup>

**Run the test on each workload separately**, not once for the whole system. A **dependency review** that follows fixed rules and a **debugging assistant** exploring unfamiliar code give different answers, so they shouldn't share a structure. Force both through one checklist, and investigations march through steps that don't apply. Give both an adaptive planner, and every dependency review pays for a decision the rules already made.

The test **isn't an argument for simplicity**; it's about **where a decision is made**. A step that needs judgement over messy input still belongs to the model. What belongs in code is the part nobody was ever unsure about: **which step comes next**.

## Five Workflow Shapes, Each Chosen by What the Task Leaves Open

The named patterns are usually taught as a catalogue. It's more useful to sort them by **what the run doesn't know when it starts**, because that's what a scenario actually tells you.<sup>[1]</sup>

| What's unknown at the start | The shape | What it gives you, and what it costs |
|---|---|---|
| Nothing; the order is fixed and each step uses the last one's output | **Prompt chaining**: one call per step | Each call has an easier job, so accuracy rises. Latency is the sum of the steps |
| Which of a known set of categories the input belongs to | **Routing**: a classifying call, then a handler per category | Each handler is tuned for one case. A wrong classification sends the request down the wrong branch |
| Nothing, and the parts don't depend on each other | **Parallelisation (sectioning)**: run parts at once, then combine | Elapsed time drops towards the slowest part. Every part is still billed |
| Nothing, but one pass isn't trustworthy alone | **Parallelisation (voting)**: run the same job several times, then reconcile | Confidence from agreement. Cost grows with attempts |
| Which subtasks the input even needs | **Orchestrator-workers**: a coordinating call splits the work, workers do it, a final pass combines | A split that fits the input. A coordinator you now have to govern |
| How many rounds it takes to meet a written standard | **Evaluator-optimizer**: one call produces, another critiques, in a loop | Measurable improvement per round, if the criteria are explicit. Needs a round limit |

Each one has a condition that selects it:

- **Chaining** needs a task that splits cleanly into **fixed** subtasks. It trades latency for accuracy.
- **Routing** needs two things: categories that **really deserve different handling**, and **classification you can trust**. Without the second, a misrouted request gets handled well by the wrong specialist.
- **Parallelisation** splits on **independence** when the goal is speed, and **repeats one job** when the goal is confidence.
- **Evaluator-optimizer** needs written criteria and evidence that another round helps. A good sign it fits: a person giving feedback measurably improves the output.
- **Orchestrator-workers** sits on the boundary of this chapter. The other four have their control flow written in code. This one **lets the model decide the split**, because the subtasks depend on the specific input. Everything later in this chapter about coordination, budgets and handoffs applies the moment you choose it.

> **Example.** A support inbox gets billing questions, bug reports and feature requests. That's a **known set of categories**, so it's **routing**: a cheap classifier (Haiku 5.5, say) sorts each message, and each category has its own prompt and tools. It isn't an agent problem, even though the messages vary a lot.

## An Agent Earns Its Cost When Later Steps Depend on Earlier Findings

Debugging an **unfamiliar system** is what autonomy was built for. The reason isn't difficulty; it's structure. Nobody can write the investigation down in advance, because the **third thing worth checking depends on what the first two found**. A stack trace points to a module. Reading the module rules out the obvious cause and suggests another. The next tool call exists only because of that result. The **path is discovered, not planned**.

That's the property to look for, and it's narrow: **a later step that exists because of an earlier finding**. Varied input alone doesn't qualify. A known set of variations is a routing problem.

The loop is short. Claude gets the request, its instructions, tool schemas and any history. It either answers or asks for a tool. The tool runs, the result goes back, and the cycle repeats until a response arrives with **no tool calls**.<sup>[2]</sup> **Real feedback from the environment** at each step is what makes the next decision better than a guess.<sup>[1]</sup> An agent with no feedback has nothing to adapt to.

The loop brings costs that exam scenarios test:

- The route is chosen by the model, not by branching logic a reviewer can read. **Normal logs aren't enough**; the interesting decisions live in a **transcript**.
- **Errors compound.** A wrong turn at step four is the input to step five.
- **The loop won't stop on its own** unless you make it.

So **exit conditions are part of the design**, and they must not depend on Claude choosing to finish. The Agent SDK has two caps that work without cooperation: a **maximum number of turns** (tool round trips) and a **maximum spend**. Each returns its own result subtype (`error_max_turns`, `error_max_budget_usd`), so your code can tell a finished task from a cut-off one.<sup>[2]</sup> Without that, a run that stopped because it **ran out of turns** looks the same as a run that **finished**, if your log only records the last message.

Two mistakes sit on either side, and both look like competence:

- **Autonomy on a settled sequence.** Someone picks an agent because agents are interesting, for a task a colleague could recite from memory. Every run pays for a planning call that decides nothing, and runs vary for no benefit. You **bought variance**.
- **A fixed checklist on an investigation.** It assumes the order of inquiry is known, which is exactly what an unfamiliar system denies. The assistant marches through irrelevant steps while the real lead goes unexplored.

Anthropic's advice on where to start: **optimise a single call** with retrieval and good examples first, and add structure only where the simpler approach **clearly fails**.<sup>[1]</sup>

## Choose Per Path, So One System Can Hold Both a Chain and a Loop

Chains and agents are often argued about as if a system must be one or the other. Most production systems are both.

The enumeration test asks about a **route through the work**, and one application often has several routes. Invoices are a clean example. Most arrive in a known shape and go through the same extraction and validation every time; a model choosing the order adds nothing. The rest arrive **malformed**, and what happens next is a real decision: send it back, ask the sender, or escalate to a person. Nobody can say in advance which applies.

Run the test on each route, and the answer is obvious: a **coded sequence for the normal route**, with a **model-driven step** only where the route **forks unpredictably**. That's not a compromise; it's the same rule applied twice.

> **Example.** In the invoice system, 95% of invoices go through `extract → validate → post` in code. When validation fails, the code calls an agent with tools like `request_correction`, `ask_submitter` and `escalate_to_human`, and lets it decide. The common path stays cheap and predictable; only the exceptions pay for judgement.

**Choosing one pattern for the whole system** makes both mistakes at once. Pick the loop, and every routine invoice pays for a planning call about something never in doubt. Pick the chain, and the exception falls through a branch nobody could write. Your **traffic logs** usually tell you how much each mistake costs.

## A Coordinator Owns the Budget, the Validation and the Contract at Each Handoff

In a **coordinator (or supervisor) setup**, one agent splits a request and **delegates pieces to specialists**. Each specialist has its own conversation, its own tools and a history nobody else reads. On **Claude Managed Agents**, every agent has its own model, system prompt, tools, MCP servers and skills, and runs in its **own isolated thread**. Tools, servers and context aren't shared.<sup>[3]</sup>

Three properties decide scenarios, and none shows up in a diagram:

- **Delegation goes one level deep** in Managed Agents. A coordinator calls agents on its roster; an agent that has its own roster can't be added, and trying fails validation.<sup>[3]</sup> In Claude Code, subagents can nest up to **three levels** by default, which you can lower, including to no nesting.<sup>[4]</sup>
- **Threads persist.** A coordinator can send a **follow-up** to an agent it used earlier, and that agent still has its earlier turns.<sup>[3]</sup> That makes review loops possible.
- **Spend is pooled.** A **session budget** is one cap shared by every thread; threads pause as it's reached.<sup>[3]</sup> In the SDK, subagent spend counts towards the total, and the next spawn fails once the cap is hit.<sup>[2]</sup>

Now the failure the exam tests hardest: **delegation without governance**. The coordinator hands out work and **accepts whatever comes back**. It looks like an architecture but behaves like a mailing list. Results come back in incompatible shapes. One worker's malformed output becomes the next stage's input. Contradictions between workers never get resolved, and the first person to notice is a user reading the final answer.

The fix is **enforcement at each handoff**, in three parts:

1. A **budget per worker**, so no single worker can use up the run.
2. **Validation** of each result against a **defined shape** before it moves on.
3. An **explicit contract** per handoff, so both sides agree what a finished subtask contains.

Three plausible alternatives fail:

- **Retries plus a shared trace** help with recovery and postmortems, but **intercept nothing** on the way downstream.
- **Letting workers negotiate** with each other deletes the component whose job was to settle contradictions.
- **Collapsing everything into one call** throws away the isolation and parallelism you built the design for.

What goes into each delegation matters too. Anthropic's research system found that short task instructions produced workers that **misread the task or repeated each other's searches**. The fix was giving each subtask an **objective**, an **output format**, guidance on **tools and sources**, and **boundaries**.<sup>[5]</sup> The coordinator's prompt also encoded **how much effort a query deserves**: one agent and a few calls for a simple lookup, many more for broad research. Without that rule, it spawned dozens of agents for questions that needed one.

> **Example.** Weak delegation: "Research competitor pricing." Strong delegation: "Find the published monthly price for the entry-level plan of Acme, Globex and Initech, as of this month. Use only each company's own pricing page. Return a JSON list of `{company, plan_name, price_usd, url}`. Don't research enterprise plans."

## Fifteen Times the Tokens of a Chat, Paid Back Only by Parallel Exploration

The published numbers aren't small. An **agent uses about 4×** the tokens of a chat; a **multi-agent system uses about 15×**.<sup>[5]</sup> Every worker holds its own context and spends its own tokens on it.

Here's what that bought. A system with a Claude Opus 4 lead and Claude Sonnet 4 workers beat a single Claude Opus 4 agent by **90.2%** on an internal research eval, and Anthropic attributes most of the difference to **token spend**.<sup>[5]</sup> The design works largely because it buys more parallel computation. That's a real gain, and it's also the whole bill.

So it only fits narrow conditions. Fan-out earns its cost on work that **really splits into independent parts**: research across many **separate sources**, material **bigger than one context window**, or **many complex tools**.<sup>[5]</sup> It doesn't fit tasks where every agent needs the **same context**, work with **heavy dependencies** between pieces, or **most coding**, where each step uses the last one's result.<sup>[5]</sup> Later Anthropic research adds that **passing context between agents costs about as much as acting on it**, so coordination is far from free.<sup>[6]</sup>

**Fan-out on a dependent chain** is the mistake, and it usually shows up as a latency fix. Slow work is split across parallel workers, on the idea that simultaneous work finishes sooner. Elapsed time barely moves, because the pieces were waiting on each other all along, and the bill multiplies.

There's also a **control cost**. Every worker is another place the run can break. Retry, backoff and fallback have to be wired into **each worker separately**. One worker stuck on a rate limit, with nothing to back off with, can hold the final step open while the coordinator waits forever.

Two levers reduce the cost without abandoning the design:

- Run the **coordinator on a capable model** and the **workers on cheaper ones**.
- Keep the roster honest: workers aimed at **truly separate territory** return mostly different findings, which is the point.<sup>[6]</sup> Overlapping workers are money spent twice.

## A Subagent Starts Empty, So Its Prompt Must Carry What It Needs

**Isolation** is what makes delegation useful, and also what breaks handoffs when a designer forgets it.

A subagent that **isn't a fork** starts with **no parent conversation**. The only thing that crosses from parent to child is the **prompt** that launched it. File paths, error text and decisions already made are only there if someone put them there.<sup>[7]</sup> In the other direction, the child's tool calls and results stay inside it, and **only its final message returns**.<sup>[7]</sup> Chapter 7 covers this as a context technique. Architecturally, the point is: **nothing crosses unless something carries it**.

**Not everything inherits the same way.**<sup>[8]</sup>

| Inherited by a Claude Code subagent | Not inherited |
|---|---|
| Permission scope | The parent's conversation |
| Project `CLAUDE.md` files | Skills the parent already used (list them in the subagent's definition) |
| Its own system prompt and preloaded skills | Files the parent already read |
| A git snapshot | Preferences stated in the main conversation (restate them in the prompt) |

So handoffs are **recorded, not narrated**. **The stage that sees a detail writes it down**, because no later stage can recover a line range nobody kept. If a coordinator collects three workers' findings and asks a final stage to "attribute each claim to its file", it's asking for facts that stage never saw, and guessing is how a claim gets attached to the wrong file. The fix is a **structured record created where the observation happens**: path, symbol, line range, and the snippet that proves it.

> **Example.** Instead of a worker returning "Found a SQL injection risk in the user lookup", have it return `{"file": "api/users.py", "symbol": "get_user", "lines": "42-47", "evidence": "cursor.execute(f\"SELECT * FROM users WHERE id={user_id}\")", "issue": "SQL injection"}`. The final report can now cite it exactly.

Two alternatives fail:

- **Concatenating transcripts** and calling it integration gives the final stage everything with no structure, so attribution gets worked out a second time, badly.
- **Regex over summaries** extracts whatever a summary happened to mention. Every omission stays omitted, and looks like "no findings".

**Ordering between stages** needs the same care. A report stage that runs before its source stage finishes produces claims nobody can check. Don't guess completion from words in the output. **Block the next stage on an explicit signal** the producing stage sets: a status plus a **structured list of what it produced**.

**Coverage** is the same argument one level up. When output reads well but **one area is thin**, more effort aimed the same way just re-samples the same blind spot, and a higher iteration count spends more without aiming it. Close the gap with an explicit **check against the intended scope**, then **targeted follow-up work** and a rerun of the final pass. Managed Agents formalise this as a defined **outcome with a rubric**: a **grader** evaluates the result in a **separate context window**, so it isn't swayed by the agent's own choices, and its feedback goes back for the next round.<sup>[9]</sup>

## Memory Is a Design Decision About What Survives the Session

When an agent loop ends, everything it gathered ends too, unless you decided otherwise at **design time**. This book calls that decision **memory scope**. The two mistakes pull in opposite directions: too much state in the conversation **inflates every call**; too little written anywhere **loses continuity**.

| Scope | What's still there next session | What it costs | What it loses |
|---|---|---|---|
| In the conversation | Nothing; survives turns, not restarts | No retrieval, but token cost grows every turn | Everything, on a new session |
| External storage | Whatever was written, across sessions, users and instances | Retrieval latency, plus read/write logic | Nothing on the storage side |
| Summarised and re-injected | A summary of earlier conversations | Less than replaying full history | Any detail the summary left out |
| Nothing kept | Nothing, by design | No overhead | Everything from earlier sessions |

The trap is that the first row **works beautifully in a prototype**. Then cost grows with session length, latency climbs, and the fix arrives at the worst moment, under production pressure. Deciding at design time costs an afternoon; deciding later costs a release.

Managed Agents give the external option a documented shape: a **memory store** is a set of documents **mounted into the session as a folder**. The agent reads and writes it with its normal file tools, and a note in the system prompt says where it is. Every change creates an **immutable version**, giving an audit trail and point-in-time recovery. Access can be **read-only** for shared reference material.<sup>[10]</sup>

**Resumability** is a different requirement. A half-hour task with clear checkpoints needs state that **outlives the process**, so an interrupted run restarts from the **last completed checkpoint**. A **longer timeout** makes interruptions rarer, not survivable. A **duplicate copy** running alongside doubles the cost and fails to anything both share. Anthropic reached the same conclusion: restarts are expensive, so pair the model's adaptability with **deterministic retry logic and regular checkpoints**.<sup>[5]</sup>

## Frameworks Wrap the Same Loop, and Cost You Visibility

**Agent frameworks** exist because everyone rebuilds the same loop. Strands, LangGraph, PydanticAI and **Anthropic's own Agent SDK** handle the parts that are identical everywhere: calling the model, defining and parsing tools, chaining calls.<sup>[1]</sup> Nobody should write those twice.

What they cost is **visibility**. Anthropic is direct about it: abstraction layers can **hide the prompts and responses** underneath, making debugging harder, and wrong assumptions about what's underneath are a common source of errors. Their advice is to **start with the API directly**, since many patterns take only a few lines. If you use a framework, understand its internals, and **reducing abstraction** for production is a legitimate move.<sup>[1]</sup>

Underneath is a different question: **who runs the loop?**

- With the **Messages API**, you own every iteration and exit condition.
- With an **SDK**, the loop runs **inside your process**, with tool execution and iteration handled for you.
- With a **hosted service** like Managed Agents, the loop and sandbox run elsewhere. The agent is a **versioned resource**; your app sends events and streams results back.<sup>[11]</sup>

Chapter 13 covers that choice. Either way, a framework decides how much of the loop someone else maintains. It has no opinion on whether the task **needed a loop at all**.

One more scale point uses the same idea. When orchestration grows beyond what one conversation can coordinate, Claude Code's **dynamic workflows** move the **plan into a script** the runtime executes. The loop, branching and intermediate results stay in the script, and the conversation only gets the **final answer**.<sup>[12]</sup> That's the enumeration test again, applied to the coordination itself.

## How the Exam Builds Wrong Answers

Wrong answers here name real architectures that competent teams have shipped, so looking for obvious nonsense finds nothing. What separates them is one assumption underneath. Four assumptions produce nearly all of them:

- **Autonomy where the sequence is already settled.** The option lets the model plan a route, in a scenario that describes the steps as fixed, repeatable or already done by hand. It's correct only when the scenario says the **path depends on what earlier steps find**.
- **Ignoring isolation, or paying for it and throwing it away.** Treating separate agents as if they share a conversation, forwarding every message to every agent, or removing the coordinator. At a handoff, the question is never whether a fact is visible; it's **what gets passed**.
- **Watching, negotiating or cleaning up instead of coordinating.** Tracing, retries and dashboards intercept nothing. Peer-to-peer negotiation removes the coordinator's job. Generating output and pruning it afterwards leaves a structure shaped by claims that never held.
- **More capacity or effort where structure is the problem.** A larger window for interleaved sub-problems that already fit. A higher iteration count for a coverage gap. A longer timeout for interruptions that need to be survived. Name the quantity the option changes, and check whether the scenario named it.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| The steps are repeatable, known, or already done by hand | A fixed sequence of prompts, one per step | Letting the model plan a route that's already decided |
| The right path is unknown and depends on what's found | Planning that adapts to each finding and carries findings forward | A fixed checklist, ignoring intermediate results, or restarting when a lead dies |
| One system has a fixed-rule workload and an open-ended one | Different structures for each, split by whether the next step is known | One structure for both, or one huge prompt for everything |
| Findings reach a later stage attached to the wrong file, or missing details | A structured record written where the observation was made | Concatenating transcripts, regex over summaries, or asking the last stage to reconstruct |
| A later stage runs before an earlier one finishes | Block it on an explicit status and output list the earlier stage sets | Guessing completion from prose, or generating first and pruning later |
| A top component interprets requests; subordinates keep their own histories and tools | A coordinator with isolated context per subagent | Reading it as one shared history, a pipeline, or an index |
| Workers return contradictory or malformed output that flows downstream | Per-worker budgets, schema validation before handoff, explicit contracts | Retries plus a shared trace, peer negotiation, or one big call |
| One request mixes several unrelated sub-problems and progress gets lost | One worker per sub-problem, with the coordinator tracking completion | A larger window, or asking the agent to keep its own tally |
| Output is coherent but one area is thin | A coverage check against the intended scope, plus targeted follow-up | Repeating the broad task, more iterations, or hiding the thin section |
| Subtasks have different success criteria and one may send work back | A state-based structure with defined transitions | A linear chain with no way back, or one loop holding every tool |
| A task runs for tens of minutes with named checkpoints | State saved at each checkpoint and reloaded on resume | A longer timeout, or a duplicate copy in parallel |
| Cost multiplied after a fan-out, and answers barely improved | Check whether the pieces were ever independent; one agent may fit | Treating parallel workers as a latency fix, or adding more |
| One flow has a routine majority and a small unpredictable remainder | Code for the routine path, with the model only where it forks | One pattern for the whole system, in either direction |

One habit does more than the table. Before comparing options, name **what the scenario says is unknown** when the run begins, and **what it says is settled**. Almost every wrong answer moves a decision to the wrong side of that line.

## What to Remember

- **Enumeration test**: can the steps be written down before the input arrives? If yes, **code holds the plan**. If no, the **model holds the plan** and **code holds the limits** (turn caps, spend caps, validation).
- **Workflows** follow **paths fixed in code**; **agents** direct their own process. Same model and tools, different owner of the next decision.
- **Five workflow shapes**, chosen by what's unknown: **chaining** (fixed ordered steps), **routing** (known categories), **parallelisation** (independent parts, or repeated attempts), **orchestrator-workers** (the split depends on the input), **evaluator-optimizer** (written criteria, and more rounds help).
- **Autonomy is justified** only when a **later step exists because of an earlier finding**. Varied input alone is a **routing** problem.
- **Agent loops don't stop on their own.** Use a **max turn count** and a **spend cap**, which return distinct error subtypes.
- A **coordinator owns** the **per-worker budget**, **validation of each result**, and an **explicit handoff contract**. Tracing and retries intercept nothing; peer negotiation removes the coordinator's job.
- **Delegated tasks need** an objective, an output format, named tools and sources, and boundaries.
- **Agents cost ~4× a chat; multi-agent systems ~15×.** That's only worth it for **truly parallel** work: separate sources, more than one window of material, many tools. Dependent work pays for fan-out and gets nothing back. Each worker needs its own retry and fallback.
- A **subagent starts empty**: the **prompt is the only way in**, and **only the final message comes back**. Skills don't carry over; permissions do.
- **The stage that sees a detail records it**: path, symbol, line range, evidence. **Sequence stages on an explicit status**, not on words in the output.
- **Decide memory scope at design time**: in-conversation (dies with the session), external storage (survives, costs retrieval), summarised (keeps only what the summary kept), or none. **Resumability needs saved checkpoints**; a longer timeout isn't the same.
- **Frameworks wrap the same loop and cost visibility.** Start with the API, understand what's underneath, and reduce layers for production.
- **Choose the pattern per path, not per system.** Code for the settled route, and a model-driven step only where the route forks unpredictably.

## Endnotes for this chapter

1. Workflows orchestrate LLMs and tools through predefined code paths, while agents dynamically direct their own processes; the five workflow patterns (prompt chaining, routing, parallelisation with sectioning and voting, orchestrator-workers, evaluator-optimizer) each with a selecting condition; start with the simplest approach and optimise single calls with retrieval and in-context examples first; agents rely on ground truth from the environment and stopping conditions such as a maximum iteration count; frameworks simplify model calls and tool handling but add abstraction that can obscure prompts and responses, incorrect assumptions about what's underneath are a common source of error, and reducing abstraction is appropriate for production. Anthropic, *Building Effective AI Agents* (Anthropic Engineering, accessed 2026-09-09; not re-checked for this edition), [https://www.anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)

2. The agent loop repeats until a response carries no tool calls; maximum turn and budget options return `error_max_turns` or `error_max_budget_usd`; subagent spend counts toward the query total, and a further spawn fails once the cap is reached. Anthropic, *How the agent loop works* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/agent-loop](https://code.claude.com/docs/en/agent-sdk/agent-loop)

3. In multiagent orchestration, each agent runs in its own thread with its own model, system prompt, tools, MCP servers and skills, none shared; threads persist, so a coordinator can follow up with an agent it used earlier; delegation is one level deep, and adding an agent that has its own roster fails validation, with up to 20 unique agents in a roster; a session budget is one cap shared across threads. Anthropic, *Multiagent orchestration* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration)

4. By default a subagent can spawn its own subagents up to three layers below the main conversation; the limit is configurable, and setting it to one turns nesting off. Anthropic, *Create custom subagents* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents)

5. Agents use about four times the tokens of chat and multi-agent systems about fifteen times; a system with a Claude Opus 4 lead and Claude Sonnet 4 workers beat single-agent Claude Opus 4 by 90.2% on an internal research eval, with token usage explaining most of the variance; the pattern suits heavy parallelisation, information beyond one context window, and many complex tools, and doesn't suit shared-context tasks, heavily dependent domains or most coding; each subtask needs an objective, output format, tool and source guidance, and boundaries; effort-scaling rules live in the lead's prompt; restarts are expensive, so retry logic and regular checkpoints let agents resume. Anthropic, *How we built our multi-agent research system* (Anthropic Engineering, accessed 2026-09-09; not re-checked for this edition), [https://www.anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system)

6. Multiagent approaches suit parallelisable problems and struggle with interdependent ones; independent parallel agents found largely different vulnerabilities, with only twelve in common; transmitting context between agents costs about as much as acting on it. Anthropic, *Patterns and problems in multiagent systems* (Anthropic Research, accessed 2026-09-09; not re-checked for this edition), [https://www.anthropic.com/research/multiagent-systems](https://www.anthropic.com/research/multiagent-systems)

7. Each subagent runs in its own conversation that starts fresh unless it's a fork; only its final message returns; the only content passed in is the Agent tool's prompt string, so paths, errors and decisions must be in it; independent subagents can run concurrently. Anthropic, *Subagents in the SDK* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/subagents](https://code.claude.com/docs/en/agent-sdk/subagents)

8. A non-fork subagent doesn't see the parent's history, invoked skills or read files; its initial context holds its system prompt, the delegation message, the project context-file hierarchy, preloaded skills from its own definition, and a git snapshot; preferences from the main conversation must be restated. Anthropic, *Create custom subagents* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents)

9. An outcome defines the end result and a rubric; a grader evaluates the artifact in a separate context window and returns its explanation for the next iteration, with results `satisfied`, `needs_revision`, `max_iterations_reached`, `failed` or `interrupted`. Anthropic, *Define outcomes* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/define-outcomes](https://platform.claude.com/docs/en/managed-agents/define-outcomes)

10. A memory store is a workspace-scoped collection of documents mounted as a directory in the session, read and written with file tools, with a note added to the system prompt; every change creates an immutable version; read-only access is supported. Anthropic, *Using agent memory* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/memory](https://platform.claude.com/docs/en/managed-agents/memory)

11. The Messages API suits custom loops and fine-grained control; Managed Agents provide a configurable runtime on managed infrastructure for long-running work, with the agent defined as a versioned resource and the application sending events and streaming results. Anthropic, *Claude Managed Agents overview* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/overview](https://platform.claude.com/docs/en/managed-agents/overview)

12. A dynamic workflow moves the plan into a script that holds the loop, branching and intermediate results, so the conversation's context holds only the final answer; it suits tasks needing more agents than one conversation can coordinate. Anthropic, *Orchestrate subagents at scale with dynamic workflows* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/workflows](https://code.claude.com/docs/en/workflows)
