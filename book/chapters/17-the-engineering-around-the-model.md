# Chapter 17: REST, Versioning, Review: the Exam Beyond Claude

**Summary**: *The third-heaviest skill on the exam barely mentions Claude. It covers REST contracts, JSON that tolerates fields it hasn't seen, concurrency you limit yourself, prompts reviewed like code, a build that checks for real regressions, and migrations that pin behaviour before changing it. If you only study the model, these show up as ordinary engineering questions with nothing model-specific to answer them from. They're worth as many marks as model questions, and there are more of them.*

## Seven Topics, None of Them About Claude

The blueprint clause this chapter covers lists: **REST APIs**, **JSON**, **asynchronous programming**, **version control**, **lifecycle integration**, **code review**, and **refactoring** at small and large scale.

None of those is a Claude topic. They're what a competent engineer does on an ordinary day, and together they're the **third-heaviest skill** on the exam.

That creates a predictable study mistake. Someone reads every page about the model, then meets a scenario about **forty call sites**, a **deprecated parameter** and a **six-month window**. The question is about **migration risk**. Claude is just the setting.

The practices don't change because a model is involved. What changes is the **dependency** they have to handle. Call it a **model-shaped dependency**: it's **remote**, **variable** in its output, **billed per call**, **versioned on someone else's schedule**, and able to **fail while reporting success**.

| Property of the dependency | What it changes about ordinary practice |
|---|---|
| Output varies between runs on identical input | A test that checks exact equality reports noise, not regressions |
| Every call costs money in proportion to its size | Token spend becomes a kind of regression, like latency |
| Versions retire on the publisher's schedule | Migration is planned engineering work, not maintenance |
| A failure can arrive as a well-formed answer | Error handling that only reads the status code misses cases |
| Behaviour is defined in prose, not control flow | Prompt text is code and lives where code lives |

Each section below is a standard practice with one of these properties fed into it.

## A Contract Is What's Promised Plus What's Reserved

Chapter 2 covers the parts of a call and chapter 3 the endpoints. The engineering question underneath is: what does this contract **promise**, and what has the publisher **reserved the right to change**?

**What's promised:**

- Claude is reached over a **REST API**. Requests and responses are **JSON**, and every response carries a **request ID**.<sup>[1]</sup>
- The service is **stateless**: it keeps nothing between calls, so continuity is something your code provides.
- Every request needs an **`anthropic-version` header**. The current version is **`2026-10-21`**, the first new one since `2023-06-01`, and Anthropic recommends using the latest version whenever possible. The SDKs set the header for you.<sup>[2]</sup>

A pinned version doesn't freeze the service. It makes an upgrade a **decision someone chose**.

> **Example.** Version `2026-10-21` removes `temperature`, `top_p` and `top_k` (any value returns a `400`) and rejects a few other request shapes. The new SDK major versions (Python 2.0.0, TypeScript 1.0.0 and others) **send `2026-10-21` by default**. So a routine `pip install -U anthropic` that crosses a major version moves every request to the new version at once. A team that pins its SDK version and upgrades on purpose finds this in a test run, not in production.<sup>[2]</sup>

**What's reserved:** errors come back as **JSON** with a **type**, a **message** and a **request ID**, and the documentation says the set of `type` values **may grow over time**.<sup>[3]</sup> That tells you how to write the consumer:

- **Branch on the values you know**, with a **default branch** for everything else.
- **Log the request ID**, not the message text.
- **Catch the SDK's typed exceptions**; don't match on message wording, which isn't part of the contract.<sup>[3]</sup>

```python
try:
    response = client.messages.create(...)
except anthropic.RateLimitError:
    retry_later()
except anthropic.APIStatusError as e:
    # Covers error types we didn't list, including new ones
    log.error("Claude API error", request_id=e.request_id, status=e.status_code)
    raise
```

The anti-pattern is the **exhaustive switch over someone else's list**. It handles every documented value and looks complete, so the reviewer approves it. But it's complete only against a list the publisher said it might extend. The first new value hits a branch that doesn't exist.

**JSON discipline** is the same rule at small scale: **read fields by name**, **ignore fields you don't recognise** instead of rejecting them, and treat a **missing field** and a **null field** as two different things.

## Concurrency Is Your Process's Job, Not the Service's

"Asynchronous" means two unrelated things on this syllabus, and scenarios keep them apart:

- A **service** that accepts a batch of work and returns results later (the Batch API, chapter 3). It buys a **discount, not speed**.
- The **programming model in your code**: starting many calls without waiting for each one. That's this section.

A thread waiting on a network call is doing nothing. **Non-blocking code** starts many calls, stays useful while they're in flight, and handles each result as it arrives. This is **overlapping waiting**, not parallel computing, so one process can hold **hundreds of calls** open without hundreds of cores.

Four consequences matter in high-volume scenarios.

**1. The limit that matters is the one you set.** Launching ten thousand calls at once hits the **rate limit as a wall**, and every rejected call burns retry budget the others need. A **bounded pool** turns a burst into a steady rate.

**2. Results come back in any order.** Each unit of work carries a **key**, and you **match results by key**, never by position in a list. Matching by position is how the wrong customer record gets updated, with no error raised.

**3. Retries and concurrency multiply.** Retry budget is **shared** across the whole process. Many workers retrying on their own, against a limit they're causing together, keep the limit hit indefinitely.

**4. A retried call must be safe to repeat.** Concurrency makes retries routine. If a unit of work writes a record, charges a card or sends a message, give it an **idempotency key** the receiver recognises. Without one, a correct client and a correct service can create the **same record twice**, and neither log shows an error.

```python
import asyncio

sem = asyncio.Semaphore(20)  # at most 20 calls in flight

async def classify(ticket):
    async with sem:
        resp = await client.messages.create(...)
        return ticket["id"], resp      # the key travels with the result

results = dict(await asyncio.gather(*(classify(t) for t in tickets)))
# results["T-1042"] is always ticket T-1042's answer, whatever order they finished in
```

The named failure is the **unbounded fan-out**. Someone writes a sequential loop, it's too slow, and the quickest fix is to launch everything at once. If the volume is high and **nobody is waiting on any single answer**, the better move is often the **Batch API**, not your own concurrency.

## Version Control Holds Everything That Decides What Ships

The test is one sentence: if changing it **changes what the system does in production**, it **belongs in the repository** with the code, tracked and reviewed the same way.

| Artifact | Where it often ends up | Why the repository is right |
|---|---|---|
| Prompt text | A string in a handler, or a database row nobody diffs | It defines behaviour; an edit is a behaviour change with no commit |
| Model ID | Inline at the call site | It fixes capability, price and a retirement date, often in several files |
| Tool and output schemas | Next to the code (usually right already) | They're the contract consumers were written against |
| Eval sets and thresholds | Someone's spreadsheet | A threshold means nothing without the fixed set behind it |
| Credentials | Wherever they were first pasted | They don't belong in the repository at all (chapter 22) |

Prompts are the row people argue about. A prompt is prose, and **prose feels like documentation**. It isn't. It's **behaviour-defining code**: the same input with two prompt versions gives two different systems. Give prompts **one home**, a **tracked history**, and a **review gate no weaker** than the code's. A hand-written change log matches reality only while someone remembers to update it.

Claude Code shows the same rule. A project's `.claude/settings.json` reaches teammates and cloud sessions **only if it's committed**. The local `.claude/settings.local.json` is **added to git excludes** the first time it's written, so it stays out.<sup>[4]</sup> Shared configuration is shared by committing it; machine-specific configuration is kept out on purpose.

The anti-pattern is the **prompt edited in production**. The sign: a team can say what the system does today but not what it did **last month**.

## A Pipeline Stage Must Pass Two Tests

Adding this work to an existing lifecycle mostly means answering: what should the **build check** that it doesn't already? (Chapter 19 covers how to build evals and tests; chapter 23 covers the phases.)

A stage earns its place only if **both** are true:

1. It **catches a regression** that would otherwise ship.
2. Its **green result survives a correct answer worded differently**.

Three stages pass both:

| Stage | Regression it catches | What keeps it green on a reworded answer |
|---|---|---|
| Scored evaluation against a versioned set | Quality dropping after a prompt, model or config change | A score threshold, not an exact output |
| Contract tests over schemas and parsers | Schemas, output shapes and consumers drifting apart | Checks on shape and fields, not wording |
| Token and cost regression check | A change that quietly multiplies tokens per request | A budget with a stated tolerance |

Two popular stages each fail one test:

- **Comparing output byte-for-byte** to a stored response fails test 2. Every correct rephrasing trips it, so it fails when nothing's wrong, and it gets **switched off within a week**. Switching it off is the rational response.
- **Automatic remediation**, where the pipeline rewrites the prompt when a score drops and deploys it unreviewed, fails test 1. It doesn't catch the regression; it **deletes the signal**, along with the human judgement the failing score was meant to call in.

> **Example.** A ticket summariser's eval set has 200 tickets. The pipeline fails the build if the average score drops below 0.85, or if average tokens per request rises more than 15%. A change that rewords summaries but keeps them accurate passes. A change that drops action items fails on score. A change that doubles the prompt length fails on tokens.

**Reproducibility matters most when the pipeline runs Claude Code itself.** `claude -p` (non-interactive) loads the same context an interactive session would, including anything in the **working directory or `~/.claude`**: hooks, skills, plugins, MCP servers, auto memory and `CLAUDE.md`. Adding **`--bare` skips all of that auto-discovery**, so every machine gets the same result and only the flags you pass take effect. **`--output-format json`** returns a result a script can branch on.<sup>[5]</sup>

```bash
claude --bare -p "Review the diff in changes.patch for security issues" \
  --allowedTools "Read" --output-format json > review.json
```

Without `--bare`, a job can pass on one runner and fail on another because of a hook in someone's home directory, and nobody thinks to look there.

Claude can also run **inside repository workflows** through the Claude Code GitHub Action: it responds to an `@claude` mention on an issue or pull request, or runs a prompt on any repository event, authenticating from a repository or organisation secret.<sup>[6]</sup>

## An Automated Reviewer Produces Findings; You Still Build the Gate

Anthropic's managed **Code Review** service shows the exact distinction scenarios test. Reviews run when a pull request opens or on each push. Findings are posted as **inline comments**, and a **check run** appears with the other checks. That check run **always completes with a neutral conclusion**, so it **never blocks a merge** through branch protection. To gate merges on findings, you **read the severity breakdown** from the check run in your own CI and decide there.<sup>[7]</sup>

The general rule: a **reviewer**, human or automated, **produces findings**. A **gate** is a **rule about which findings stop a change**. They're separate. A team that installed the first and assumed it had the second has a review nobody has to act on.

Now the reviewing itself. A teammate opens a pull request that adds a Claude feature. The code works, but **three things are missing**:

- The **prompt and model are hardcoded** inline.
- There's **no error handling** for the API.
- There are **no tests** for the integration.

It's tempting to rank them and let the smaller two through. They don't rank, because **each fails in production on its own**:

- **Hardcoded config:** the prompt and model can't change without a deploy, and they aren't reviewed as things that change.
- **No error handling:** the first rate limit, timeout or overload becomes an unhandled exception, from a dependency that will certainly return errors.
- **No tests:** nobody can show later that the other two were ever fixed.

So the review **asks for all three before merge**. Each partial answer lets one through:

- **Approving because it works** ships all three gaps.
- **Deferring error handling** to a later release holds back the gap that breaks while users are on it.
- **Approving and promising to fix it yourself** looks generous, but it removes the point of the review, moves the work instead of resolving it, and teaches the author nothing. The next pull request repeats it.

What makes any change **reviewable**: it **does one thing**, an **example shows it running**, a **test proves the behaviour**, and a **short note says what it assumes** about its environment. A reviewer who has to reconstruct the author's intent reviews it last.

## Refactoring Splits by Whether Behaviour Must Be Pinned First

**Small-scale refactoring changes structure, not behaviour**: extracting a hardcoded value into config, naming a seam, splitting a function that grew a second job. The test is that **nothing observable changed**, which is why it's safe and cheap to review.

That's also why **restructuring and behaviour changes go in separate commits**. Mixed together, nobody can later tell which half caused a problem. Separating them costs one extra commit and lets you **revert half the work**.

**Extracting configuration** is the small refactor this material asks for most. It fixes the review gap above, and it's needed for migrations: a value you can change without a deploy can be changed **one environment at a time**.

**Large-scale refactoring** is a different activity. The exam's favourite example: a **deprecated parameter**, used in **dozens of places**, with a **deadline**.

The deadline is real. Anthropic gives at least **60 days' notice** before a public model retires, **requests to a retired model fail**, and a **usage audit** helps you find where the old thing is still called.<sup>[8]</sup>

But the deadline isn't the risk. **Semantics** are. Two documented migrations show why:

- **Manual extended thinking** with `budget_tokens` returns an error on the newest models, and there's **no direct replacement**: thinking is **adaptive**, and the **effort** parameter is a separate output-level control, not a thinking budget.<sup>[9]</sup>
- **Sampling parameters** (`temperature`, `top_p`, `top_k`) return a `400` on Claude 4.7 and later when set to non-default values, and are **removed entirely** in API version `2026-10-21`. The Python SDK (from v1.0) dropped them from its types, so passing them raises a `TypeError`. The replacement is **prompting**, which is a different mechanism.<sup>[8]</sup>

In both cases **the replacement isn't a rename**. It acts on something else, so a find-and-replace across forty call sites **changes behaviour at every one** and reports nothing.

That fixes the order of work:

1. **Pin current behaviour with regression tests**, while the old parameter still works. Afterwards there's nothing left to observe.
2. **Migrate in small batches**, small enough that a failure points to one batch.
3. **Validate each batch** against the tests before starting the next.

> **Example.** Forty call sites set `temperature: 0.2` to keep a classifier's labels consistent. Before migrating, record the labels on a fixed set of 300 inputs. Remove the parameter from five call sites, add an instruction and a structured output schema with an enum of labels, rerun the set, and compare. Only then do the next five.

Two named failures, both answers to a question nobody asked:

- The **compatibility wrapper** keeps the old function signature while the meaning changes underneath. Every call site compiles, the diff is tiny, and the semantic change is hidden in one function no test watches. The parameter still disappears on schedule.
- The **deadline-day migration** schedules the work for the end of the window, putting every semantic change into one unvalidated push with no time left to recover.

Both trade a real risk for a schedule that feels tidier.

## How the Exam Builds Wrong Answers

Wrong answers here are hard to rule out because they describe things competent teams really do. Four assumptions produce most of them.

- **Schedule risk instead of semantic risk.** The option deals with **when** work happens, not whether the change is **safe**: a wrapper, a batching plan, a later release. Quick test: would it still be the right answer if the deadline were a year away? If yes, it never addressed the risk. It's correct only when the replacement really is a drop-in rename.
- **A check with no tolerance for rewording.** Exact string equality, a fixed response length, a stored output compared literally. Each fires when the answer is correct but phrased differently, so it gets disabled. Ask what a **correct, reworded answer** does to the proposed check.
- **Judgement removed where judgement was the point.** Auto-rewriting a prompt when a score drops; approving a change with a promise to fix it privately; treating an automated reviewer as a merge gate. Ask which human was supposed to decide, and whether the option leaves them anything to decide with.
- **A partial fix accepted because it's the biggest.** Three independent gaps, and the option closes the largest. It's a real improvement, which is why it survives a first read. If each gap fails on its own, the answer is **all of them**.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| A parameter is deprecated and its replacement behaves differently | Pin current behaviour in tests, then migrate in attributable batches | A wrapper that keeps the call shape, or one swap at the deadline |
| A pipeline is being designed for a Claude-backed service | Stages that catch real regressions and stay green on rewording | Literal output comparison, or auto-rewrite and deploy |
| A change hardcodes config, skips error handling and has no tests | Ask for all three before merge | Approving and fixing it yourself, or deferring error handling |
| A scripted job must behave the same on every machine | Load nothing from the host (`--bare`); pass context explicitly | Assuming a scripted run sees what a developer's session sees |
| Prompts are edited by hand and nobody knows what changed when | One home, tracked history, review like code | A hand-kept change log, or optional review for prose |
| High volume, independent units, nobody waiting on any one result | Bounded concurrency with a key per unit, or the Batch API | Launching everything at once, or a slow sequential loop |
| A retry fired and the same action happened twice | Make the work safe to repeat with an idempotency key | More retries, fewer retries, or backoff |
| An automated reviewer comments on every pull request | Read its findings in your own CI and decide what blocks a merge | Treating the reviewer's presence as the gate |
| A response field gains a value the code didn't expect | Branch on known values and default the rest | An exhaustive switch, or matching on message text |
| An SDK upgrade suddenly causes `400` errors | Check whether the new SDK major sends a newer `anthropic-version` | Rolling back blindly, or assuming the service broke |

One habit works beyond the table: for each option, name which of the **five properties** it accounts for, and which it **quietly assumes away**. Wrong options are usually correct engineering aimed at a dependency that behaves better than this one.

## What to Remember

- A **model-shaped dependency** **varies between identical runs**, **bills per call**, **retires on the publisher's schedule**, **can fail while returning success**, and has **behaviour defined in prose**.
- A **contract** is **promised plus reserved**. Error `type` values **may grow**: branch on known values, default the rest, catch **typed exceptions**, never match message text.
- **`anthropic-version` is required.** The current version is **`2026-10-21`**; new SDK majors send it by default, so an SDK upgrade can change API behaviour. Pinning makes an upgrade a decision.
- **Concurrency is overlapping waiting.** **Bound it yourself**; match results **by key**, not position; share retry budget; make writes **safe to repeat** with an idempotency key.
- High volume with nobody waiting → consider the **Batch API**.
- Anything that **changes production behaviour** goes in **version control**: prompts, model IDs, schemas, eval sets and thresholds. **Not credentials.**
- A **prompt is code**: one home, tracked history, review as strict as code.
- A pipeline stage must **catch a real regression** and **stay green on a reworded correct answer**. Good stages: **scored evals with a threshold**, **contract tests**, **token/cost checks**. Bad: **exact-match tests**, **auto-remediation**.
- Scripted Claude Code runs should use **`--bare`** so nothing from the host loads; use **`--output-format json`** for scripts.
- A **reviewer produces findings; a gate decides**. Managed Code Review's check run is **always neutral** and never blocks a merge on its own.
- Hardcoded config, no error handling, no tests: **ask for all three**. Fixing it yourself moves the work instead of resolving it.
- **Separate commits** for restructuring and behaviour changes.
- **Deprecations:** at least **60 days' notice**; retired models fail. A replacement that acts on something else **isn't a rename**: **pin behaviour in tests first**, then migrate in **small batches**. Avoid **compatibility wrappers** and **deadline-day migrations**.

## Endnotes for this chapter

1. The Claude API is a RESTful API at `https://api.anthropic.com`, uses JSON request and response bodies, and returns a `request-id` response header carrying a globally unique identifier for each request. Anthropic, *API overview* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/api/overview](https://platform.claude.com/docs/en/api/overview)

2. API requests must send an `anthropic-version` header, for example `anthropic-version: 2026-10-21`, and the client SDKs set it automatically; Anthropic recommends the latest version whenever possible. Version 2026-10-21, released 21 October 2026, is the first new version since 2023-06-01; it removes `temperature`, `top_p` and `top_k` (a 400 on every model, even at default values), rejects `document` blocks in assistant messages and non-empty `citations` on text blocks in user messages, and renames several Admin API fields. New SDK majors (Python 2.0.0, TypeScript 1.0.0 and others) send 2026-10-21 by default. Anthropic, *API versions* and *Migrating to API version 2026-10-21* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/api/versioning](https://platform.claude.com/docs/en/api/versioning); Anthropic, *Release notes* (accessed 2026-11-27), [https://platform.claude.com/docs/en/release-notes/api](https://platform.claude.com/docs/en/release-notes/api)

3. Errors are returned as JSON with a top-level `error` object containing a `type` and `message`, plus a `request_id`; under the versioning policy these values may expand and the set of `type` values is expected to grow; the official SDKs raise typed exceptions, and the guidance is to catch those classes rather than match on message strings. Anthropic, *Claude API errors* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/api/errors](https://platform.claude.com/docs/en/api/errors)

4. A project's `.claude/settings.json` reaches teammates and cloud sessions only when committed; `.claude/settings.local.json` is added to the user's global git excludes the first time Claude Code writes it. Anthropic, *Claude Code settings* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings)

5. `claude -p` runs non-interactively and loads the same context an interactive session would, including anything configured in the working directory or `~/.claude`; `--bare` skips auto-discovery of hooks, skills, custom commands, subagents, plugins, MCP servers, auto memory and CLAUDE.md, and is recommended for CI and scripts where the same result is needed on every machine; `--output-format json` returns structured JSON with the result, session ID and metadata. Anthropic, *Run Claude Code programmatically* (Claude Code Docs, accessed 2026-11-27), [https://code.claude.com/docs/en/headless](https://code.claude.com/docs/en/headless)

6. The Claude Code GitHub Action runs Claude Code inside a repository's workflows, responding to an `@claude` mention in a pull request or issue comment, or running on any GitHub event when given a prompt, authenticating from a repository or organisation Actions secret. Anthropic, *Claude Code GitHub Actions* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/github-actions](https://code.claude.com/docs/en/github-actions)

7. Code Review posts findings as inline comments and populates a Claude Code Review check run; the check run always completes with a neutral conclusion (or skipped when a push changed no files), so it never blocks merging through branch protection; to gate merges on findings, read the severity breakdown from the check run output in your own CI. Anthropic, *Code Review* (Claude Code Docs, accessed 2026-11-27), [https://code.claude.com/docs/en/code-review](https://code.claude.com/docs/en/code-review)

8. Once a model is deprecated, all usage must be migrated before the retirement date, and requests to retired models fail; Anthropic gives at least 60 days' notice before retiring publicly released models; an audit of API usage helps locate remaining usage. `temperature`, `top_p` and `top_k` return a 400 when set to non-default values on Claude 4.7 and later models, and are removed in API version 2026-10-21; the recommended replacement is to omit them and guide behaviour with prompting; the Python SDK (v1.0 and later) removes them so passing them raises a `TypeError`. Anthropic, *Model deprecations* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/model-deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations)

9. Manual extended thinking with `budget_tokens` is not supported on the newest models and returns a 400; `budget_tokens` has no direct replacement because thinking is adaptive, and effort is a separate output-level control. Anthropic, *Migration guide* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/about-claude/models/migration-guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)
