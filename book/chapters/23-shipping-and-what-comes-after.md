# Chapter 23: Deploy Is a Phase Boundary, Not the End

**Summary**: *A release date marks the end of development and nothing else. What starts at go-live runs until the system is switched off, and it has three jobs the build phase never had: watching the numbers, responding when they move, and planning around dependencies that change on schedules your team doesn't set. This chapter covers the phases a Claude application moves through and the gates between them, where a decision to proceed is made and, sometimes, refused.*

## Go-Live Closes One Calendar and Opens One With No End Date

The **release goes out**. Internal teams start using it that week. The **board empties**, the build channel goes quiet, and the people who wrote it are **assigned elsewhere**.

Something really did end. Development had a **scope**, a **definition of done** and a **date**, and all three were met. What started in the same hour has **none of the three**, and it lasts as long as the system is in use.

Call it the **standing phase**: it **opens at go-live**, covers **operation and maintenance**, and has **no completion criterion**, because a system people depend on always needs to be **watched, answered for and moved forward**.

A scenario will put you at exactly that moment and ask what the **next stage of work** is. Every option will describe something real:

- One names a **single activity** that does happen after launch (a retrospective, a launch review).
- One describes **who owns** the system, not what happens to it.
- One skips to the **end of its life**.
- Only one **runs as long as the system does**. That's the answer.

## A Gate That Can't Refuse Anything Isn't a Gate

Between any two phases is a **decision to move forward**. It's only worth having if you're allowed to say **no**, and "no" is only possible when **something concrete is being checked**.

That's also why gates get skipped. A checkpoint **costs a day** the schedule didn't plan for, and that day is **visible**, while what it would have caught **isn't**.

| What the gate checks | What it refuses to let through | What ships when nobody checks |
|---|---|---|
| A constraint with no named source | A platform chosen out of habit | A deployment nobody can defend at review |
| A build the current eval has never scored | Promotion to production traffic | A regression nobody can trace |
| A model reference that changes without a record | An unrecorded pin | An upstream change nobody logged |
| Metrics not emitting; no owner on the alert | Go-live itself | A failure first reported by a user |
| A production finding not linked to the requirement it contradicts | Closing the ticket | Knowledge held by whoever was on shift |

The right-hand column is the real subject. A **skipped gate** ships a system that **works on the day** and **can't be reasoned about months later** by people who weren't in the room.

The anti-pattern is **deployment treated as the finish line**. It isn't laziness. It's a calendar built by people whose **work really does finish**, handed to people whose work doesn't.

## The Standing Phase Has Three Jobs, and Only One Is Debugging

### 1. Instrument from day one

From the first day of production traffic, **record three things**: **cost per call**, **latency**, and **failure rate**.

**Cost telemetry is an estimate.** Claude Code's exported cost metric is documented as an approximation; **official billing data** comes from the API provider (the Claude Console, Bedrock or Google Cloud), not the exporter.<sup>[1]</sup> Use it for **trends and alerts**, not to settle a finance question.

**Failure counts under-report**, and in the direction that looks healthy. A **refusal** (`stop_reason: "refusal"`) arrives on a **successful response**, not as an HTTP error, so the **API error event doesn't fire** for it. A **separate refusal event** exists so you can track refusal frequency.<sup>[2]</sup> An error rate built only from transport failures stays **flat** while a growing share of responses give the caller **nothing usable**.

> **Example.** A document assistant's dashboard shows a steady 0.3% error rate for a month. Meanwhile refusals on one document category climb from 1% to 9% after a content change upstream. Users complain; the dashboard says everything's fine. Counting refusals as their own metric would have shown the trend in the first week.

The same applies to your own API code: check **`stop_reason`** on every response, not just the HTTP status.

### 2. Someone answers when a number moves

Instrumentation with **no name attached** to its alerts is a dashboard, and a dashboard only records what went wrong. Every alert needs an **owner**.

### 3. Planned evolution

This is the job most often **under-planned**, because nothing in a deployed configuration **looks** like it's moving.

## Your Dependencies Change on Someone Else's Calendar

Most dependencies stay still until you move them. A model doesn't.

The pin is real, and it's worth knowing exactly what it gives you. A **model ID identifies a pinned version**, and the model stays the same **for the life of that ID**. The guarantee covers **IDs**, not the **convenience aliases** the API accepts for some older models.<sup>[3]</sup> That keeps **behaviour steady**. It doesn't keep the model **available**.

- Anthropic **retires older models** as newer ones launch, gives customers with active deployments **at least 60 days' notice** for publicly released models, and **requests after the retirement date fail**.<sup>[4]</sup>
- **Partner platforms set their own retirement dates**, so a model's status on **Amazon Bedrock** or **Google Cloud** can differ from Anthropic's.<sup>[4]</sup> If you deploy on one and read the other's calendar, you have an expiry date nobody on the team has seen.

> **Example.** On 30 September 2026, Anthropic notified developers using Claude Sonnet 4.5 of its upcoming retirement on the Claude API.<sup>[4]</sup> A team that pinned `claude-sonnet-4-5-20250929` and had an owner watching deprecation notices put "migrate summariser to current Sonnet" in the next sprint, with time to run evals. A team that didn't found out when requests started failing.

**The API itself moves too.** In October 2026, Anthropic released API version **`2026-10-21`**, and new SDK major versions send it by default, so a routine dependency upgrade can change request behaviour (chapter 17).

The **dependency assumed frozen** causes a specific failure: a migration **run as an incident**, on a deadline set elsewhere, in the week it becomes urgent. The alternative is a **planned backlog item with an owner and a date** inside the notice window.

Migration between current models is **mostly drop-in**. Check a narrow list: **features no longer available**, **changed thinking behaviour**, and **refusal behaviour** that differs between models.<sup>[5]</sup> Whether the replacement holds up is answered by an **eval suite scored against the outgoing version's baseline** (chapter 19). Pinning is chapter 18. A team with neither can't say what a forced upgrade cost them.

## A Finding That Never Reaches the Requirements Is Just Folklore

Production teaches you things the test set couldn't: which **inputs actually arrive**, which instruction **people read differently** than intended, where the **cost concentrates**.

Those findings have one useful destination, and the **incident channel isn't it**. Iteration means a production finding **goes back to the requirement it contradicts** and re-enters the cycle as a **change with a gate in front of it**. Left where it landed, it's knowledge held by **whoever was on shift**, and it lasts until that person changes teams.

> **Example.** On-call notices that customers often paste screenshots instead of text, and the bot ignores them. Logged as "known issue" in the incident tracker, it's forgotten in a month. Routed back as a requirement ("accept image attachments and extract order numbers from them"), it gets an eval case, a design change, and a gate before it ships.

The end of the system's life has an obligation that **outlives the switch-off**. Retention runs on its own clock: **local Claude Code and Cowork session transcripts** are kept for **six years by default**, or the organisation's configured finite retention period, and **Activity Feed data** is kept for six years.<sup>[6]</sup> Decommissioning **stops the traffic** and **settles nothing already stored**.

## How the Exam Builds Wrong Answers

Lifecycle distractors are **very reasonable**. Almost every one names something a competent team really does, often in the same week as the scenario. What makes one wrong is a **category error** about what kind of thing it is.

- **A single activity promoted to a phase.** A launch review, a retrospective, a load test, a handover document: all real, all **finite**. A phase is a **container** for many activities over an **open-ended** period. The tell: a **completion date** offered as the answer to where the system now lives. It's right when the scenario asks **what to schedule next week**.
- **Ownership offered as lifecycle progress.** Handing the system to a platform team, splitting it between two groups, hiring for it. These answer **who** does the work; the work itself doesn't change. Right when the question is about **responsibility, escalation or staffing**.
- **The dependency assumed frozen.** Any option that treats the deployed configuration as **stable ground**, with change arriving only when the team chooses. Retirement dates, API versions and provider schedules move under a system **nobody touched**. Right only where the version in use is under the team's **sole control**.
- **A phase skipped because its evidence isn't uncomfortable yet.** Shipping before an eval scores the build, going live before metrics emit, closing a finding without routing it. Each is defensible in the moment; each removes an answer someone needs later.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| A system has just gone live; what's the next stage of work? | The ongoing phase: watch it, answer for it, plan its evolution | A finite activity, a staffing change, or decommissioning |
| A new version is ready and production is next | Score it against the baseline before promoting | Promoting first and watching production for differences |
| Nobody knows what a live system costs or how often it really fails | Emit cost, latency and failures; check what the failure count includes | Reasoning from the last load test, or a count that omits refusals |
| A deprecation notice, retirement date or provider schedule appears | Planned work with an owner and a date inside the notice window | Waiting until the date forces an emergency migration |
| Something learned in production needs to go somewhere | Back to the requirement it contradicts, as a gated change | Logging it in an incident record and calling it done |
| A checkpoint is being skipped to hit a date | Name what it would have refused | Accepting the schedule argument, which always sounds stronger |
| The system is being switched off | Stop traffic and handle stored data under its retention rules | Assuming decommissioning deletes everything |

One habit works beyond the table: ask of each option **whether it ends**. An **activity finishes**, an **ownership arrangement holds steady**, decommissioning **closes the file**. Only one thing on a lifecycle list **runs as long as the system does**, and when a scenario has just described a launch, that's what it's pointing at.

## What to Remember

- The **standing phase** opens at **go-live**, covers **operation and maintenance**, and has **no end** until the system is switched off. Development finishes; this doesn't.
- A **gate** is only real if it can **refuse**, and it refuses against something concrete: a **sourced constraint**, an **eval score against a baseline**, a **recorded pin**, **live metrics**, a **routed finding**.
- **Skipping a gate raises no error.** It ships a system that works today and can't be reasoned about later.
- **Instrument from day one**: **cost per call**, **latency**, **failures**. Cost telemetry is an **estimate**; billing comes from the provider.
- **Refusals aren't HTTP errors.** They arrive on successful responses, so **track refusals separately** and check **`stop_reason`**.
- Every alert needs an **owner**.
- A **model ID pins behaviour, not availability.** Models retire with **at least 60 days' notice**; requests after retirement **fail**. **Partner platforms have their own dates.** The **API version** moves too.
- Plan migrations as **backlog items with an owner and a date** inside the notice window. Migration is **mostly drop-in**: check removed features, thinking behaviour and refusals, and **score against the old baseline**.
- **Production findings go back to the requirements** as gated changes. Left in the incident log, they're folklore.
- **Retention outlives the application**: transcripts and Activity Feed data are kept for **six years by default**. Switching off **stops traffic, not storage**.

## Endnotes for this chapter

1. The exported cost metric supports trend tracking and spend attribution, but cost metrics are approximations, and official billing data comes from the API provider (Claude Console, Amazon Bedrock, or Google Cloud). Anthropic, *Monitoring* (Claude Code Docs, accessed 2026-09-10; not re-checked for this edition), [https://code.claude.com/docs/en/monitoring-usage](https://code.claude.com/docs/en/monitoring-usage)

2. The API error event is logged when a request fails and carries an HTTP status code; a refusal returns `stop_reason: "refusal"` on a successful response stream, so the API error event doesn't fire, and a separate refusal event lets refusal frequency be tracked. Anthropic, *Monitoring* (Claude Code Docs, accessed 2026-09-10; not re-checked for this edition), [https://code.claude.com/docs/en/monitoring-usage](https://code.claude.com/docs/en/monitoring-usage)

3. Each Claude model ID identifies a pinned version, constant for the lifetime of that ID; the guarantee covers model IDs, not the convenience aliases the Claude API accepts for some earlier models. Anthropic, *Model IDs and versioning* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)

4. Anthropic retires older models as newer ones launch, gives customers with active deployments at least 60 days' notice before retiring publicly released models, and requests to retired models fail; on 30 September 2026 Anthropic notified developers using Claude Sonnet 4.5 of its upcoming retirement on the Claude API; partner-operated platforms set their own retirement schedules. Anthropic, *Model deprecations* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/model-deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations)

5. Migration to current models is mostly drop-in, using the same Messages API and tool use patterns; the key things to check are features no longer available, thinking output, and safety classifier refusals where the target model has them. Anthropic, *Migration guide* (Claude Platform Docs, accessed 2026-09-10; not re-checked for this edition), [https://platform.claude.com/docs/en/about-claude/models/migration-guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)

6. Local session transcripts from Cowork and Claude Code are stored for six years by default, or for an organisation's finite custom retention period, and Activity Feed data is retained for six years. Anthropic, *API and data retention* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/manage-claude/api-and-data-retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention)
