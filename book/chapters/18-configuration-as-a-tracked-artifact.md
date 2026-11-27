# Chapter 18: An Unpinned Version Is an Untracked Change

**Summary**: *Something regressed, three things changed since the last good state, and none of them left a commit. That's the whole subject. This chapter makes every input to your system a value someone chose and wrote down: the canonical model ID, a prompt version a rollback can reach, plugin dependencies held in a tested range, and each setting in the file whose reach matches who needs it. Get it wrong and you pay later, with a regression nobody can trace and nothing to roll back to.*

## Nobody Can Name the Change That Caused the Regression

A **summarisation service** starts returning output the **downstream parser can't read**. It worked in March. The **release log is empty** for six weeks, so as far as anyone can see, **nothing changed**.

**Three things changed** anyway:

- Someone **edited the system prompt** where the running service reads it, to fix a formatting complaint.
- A **plugin took a new release** from its marketplace overnight.
- The model name in the call was a **convenience alias**, not a fixed ID, so it started pointing to a newer snapshot.

Each one changes the output. None left a commit. So the incident review starts with **four people guessing**, and the guess can't be settled: there's no earlier state to restore and no record of which change came first.

Call this the **unattributable window**: the time between the **last state you can reproduce** and now, containing every **change nobody wrote down**. Configuration management **keeps that window empty**. It does nothing for you on a normal day; it's the only thing that helps on the day something breaks.

Anything that decides what your system produces is either **held at a value someone chose**, or it's an **undeclared variable**. Five things commonly move without a commit:

| What moves | How it moves with no commit | What holding it looks like |
|---|---|---|
| The model | A convenience alias points to a newer snapshot | The canonical model ID, in a tracked file |
| The prompt | Edited where the running system reads it | A named version, reviewed, with the previous one kept |
| A plugin | Auto-update takes the newest release | A declared version range, resolved from a tag |
| Team configuration | The file exists on one machine only | Committed next to the code it governs |
| The measurement | The eval set stays fixed while real traffic changes | A dated set, rebuilt from current traffic |

## A Model ID Names One Snapshot; an Alias Names Whatever's Newest

Every **Claude model ID** identifies a **pinned version**, and the model behind it **doesn't change** for the life of that ID.<sup>[1]</sup>

- **Claude 4.6 and later** use **dateless IDs** such as `claude-sonnet-5-5`. The dateless ID **is the canonical ID**: one fixed snapshot, **never updated in place**. A newer version ships under a **new ID**. Anthropic calls the belief that dateless IDs point to "the latest" a **common misconception**.
- **Models before 4.6** carry a date, such as `claude-haiku-4-5-20251001`. On the Claude API they also have **shorter aliases** (like `claude-haiku-4-5`) that point to the **most recent dated snapshot** for that minor version. Those aliases **can move**.<sup>[1]</sup>

So the rule is: **write the ID the documentation calls canonical**, not a shorter alias beside it. Check the model page rather than relying on memory; the format has **already changed once**.

> **Example.** `model="claude-haiku-4-5"` is an alias for an older model and could resolve to a different snapshot later. `model="claude-haiku-4-5-20251001"` is pinned. For current models, `model="claude-sonnet-5-5"` is already the pinned snapshot; there's nothing shorter to use.

Don't confuse the **model ID** with the **API version**. The `anthropic-version` header (chapter 17) fixes the **request and response contract**. The model ID fixes the **model**. Pinning one doesn't pin the other.

Two limits on what pinning buys:

- **Weights are fixed per ID, but the serving infrastructure isn't.** Routing, safety classifiers and sampling logic can change, and occasionally cause small differences in behaviour on an ID that hasn't moved.<sup>[1]</sup> Pinning **narrows** what could have changed; it doesn't make the list empty. Keep **measuring**.
- **Every model ID has its own retirement date**,<sup>[1]</sup> and **cloud partners set their own dates**, so the same model can have a different lifecycle status depending on where it runs.<sup>[2]</sup> A pin is a **dated commitment**. Chapter 17 covers the migration work when the date arrives.

The anti-pattern is the **convenience alias in production**. It's the shortest thing to type and it works the day you type it. The problem shows up when the output shape changes: the **previous snapshot was never recorded**, so there's nowhere to go back to, and the fix becomes a hotfix to your parser. **Keeping the previous ID available** is half the value of pinning, and it's the half teams skip.

## A Prompt Version Is What an Incident Timeline Can Point To

Chapter 17 covers where prompt text lives and how it's reviewed. A **version** adds a name, and that name does two jobs:

- It lets an **incident timeline** say which text produced the output under investigation.
- It gives a **rollback** a destination.

Without it, both become guesswork through editor history and memory.

**Promotion** is where a version earns its keep. A new prompt version should reach production the way a new model does: it runs on **part of the traffic**, gets **scored against the current version's baseline**, and is promoted or reverted **on that score**. Anthropic's guidance: success criteria must be **specific and measurable**, and grading methods trade speed for nuance. **Code-based grading** is fastest and most reliable; **model-based grading** is flexible once you've tested its reliability; **human grading** is most flexible and slowest.<sup>[3]</sup> (Chapter 19 covers building the set.)

> **Example.** Prompt `summary-v14` scores 0.88 on the eval set. `summary-v15` goes to 10% of traffic and scores 0.84 on the same set. It's reverted to v14 in one config change, and the incident note says "v15, reverted" instead of "someone changed the prompt last Tuesday".

A prompt with **several consumers** has **several sets of expectations**. Rolling out without checking each one is a bet that none depends on what changed. Three near-misses:

- **Telling downstream owners** a change is coming puts it on their calendar but **finds out nothing** about its effect.
- A **clean staging run** only tests what staging covers.
- **Shipping to some consumers now and the rest later** leaves one application running **two prompt versions** at once, which is a second unattributable difference.

The anti-pattern is the **prompt with no version number**. The sign: a team can show you the current text but not the previous one.

## Four Configuration Surfaces, Chosen by Reach

The rule: **put a setting in the file whose reach matches the people it must be true for.**

| File | What it holds | Who it reaches |
|---|---|---|
| `./CLAUDE.md` or `./.claude/CLAUDE.md` | Project instructions | Everyone who clones the repo |
| `CLAUDE.local.md` | One person's instructions for their copy | One machine; keep it out of commits |
| `.claude/settings.json` | Permissions, hooks, telemetry, enabled plugins | Teammates and cloud sessions, once committed |
| `.claude/settings.local.json` | One person's exceptions in one project | One machine; automatically git-excluded |
| `.mcp.json` in the project root | MCP servers the project needs | Everyone who clones the repo |
| `managed-settings.json` (or console equivalent) | Organisation policy | Every machine the organisation deploys it to |

These reaches are **documented**:

- `.claude/settings.json` reaches teammates and cloud sessions **only once committed**; `.claude/settings.local.json` is **added to git excludes** the first time Claude Code writes it.<sup>[4]</sup>
- Project `CLAUDE.md` travels through version control, so it should hold **standards, not personal preferences**.<sup>[5]</sup>
- Of the three MCP server scopes, only **`.mcp.json` in the project root** reaches anyone else. **Local** and **user** scope both write to `~/.claude.json`.<sup>[6]</sup>
- Organisation policy goes to a system path on every managed machine, or via MDM or the console.<sup>[7]</sup>

**When the same key is set in several files**, the highest level wins, in this order: **managed** → **command line** → **project local** → **shared project** → **user**. **List keys** such as `permissions.allow` **merge** across files instead of replacing each other.<sup>[4]</sup>

> **Example.** Your team needs a Postgres MCP server in every session. Alice set it up with `claude mcp add` at the default scope, so it lives in her `~/.claude.json`. It works perfectly for her and doesn't exist for anyone else. The fix is `claude mcp add --scope project ...`, which writes it to `.mcp.json`, then committing that file. A README step saying "everyone run this command" is the wrong fix: correctness then depends on everyone doing it identically.

The anti-pattern is the **shared dependency in personal configuration**, and the person who created it can't see the problem. The reverse is just as wrong: a personal experiment **committed to the project file** is now everyone's.

One more point: instruction files **load into the context window** at the start of every session and are **context, not enforced configuration**.<sup>[5]</sup> A rule there **guides** Claude and **guarantees nothing**. If a scenario needs a guarantee, it needs a hook or permission rule.

## A Plugin Dependency Follows the Newest Release Until You Constrain It

By default, a plugin dependency **tracks the latest version**, so an upstream release can change what runs under your plugin with no warning.<sup>[8]</sup> A **version constraint** holds it in a **tested range** until you decide to move.

How it works:<sup>[8]</sup>

- Constraints go in the **`dependencies` array** of the plugin's **`.claude-plugin/plugin.json`**.
- They **resolve against git tags** on the dependency's repository, named **`{plugin-name}--v{version}`**.
- If several plugins constrain the same dependency, their **ranges intersect**, and Claude Code picks the **highest version that satisfies all of them**. Auto-update still works, but only inside that range.

Which plugins are switched on is a separate setting, **`enabledPlugins`**, keyed as `plugin-name@marketplace-name`, in any settings file.<sup>[9]</sup>

With **no recorded version**, three problems share one cause: you can't find where a regression came in, you can't hold the working version still, and you have nothing to roll back to. The fix has to **create the record**.

The confident wrong answer is the **upgrade schedule treated as a version record**. Upgrading everything monthly feels like control, but it **increases unrecorded change** and writes nothing down. Waiting for authors to announce breaking changes fails the same way: the record is in someone else's hands.

## Environments Differ on Purpose; Write the Differences Down

Development, staging and production running **different model IDs, prompt versions and plugin sets** isn't a problem in itself. Staging exists to hold something production doesn't. The problem starts when **nobody can list the differences**, because then a deliberate difference and an accident look the same.

Keep **one tracked file** with the **same keys for each environment**. Every difference is then either in the file, where someone chose it, or it's **drift**.

```yaml
# config/environments.yaml
production:
  model: claude-sonnet-5-5
  summary_prompt: summary-v14
  plugins: { review-tools: "~2.3.0" }
staging:
  model: claude-opus-5-5         # testing the larger model before promotion
  summary_prompt: summary-v15
  plugins: { review-tools: "~2.4.0" }
```

Three repairs sound right and aren't:

- **Making all environments identical** erases differences that exist for a reason, and staging's purpose with them.
- **Unpinning until they agree** removes a control and puts behaviour changes outside version control again.
- **Moving the values into application code** ties every configuration change to a release, though config and code change on different schedules.

## Restore Measurability Before Improving Anything

A common scenario: a feature shipped a year ago and has run ever since. The people who built it have moved on, **prompts were edited in place** several times, the **eval set** hasn't been touched since launch, and the **model ID was never pinned**. Someone proposes **upgrading to a newer model**.

That's the **wrong first move**: it adds a change to a system whose **current behaviour was never recorded**, so the result is as untraceable as everything before it. The right order:

```
1  Pin the moving inputs       canonical model ID, plugin ranges, config in one tracked place
2  Version the prompts as they are now, whatever anyone thinks of the wording
3  Rebuild the eval set from recent production traffic, not launch assumptions
4  Change one thing, and compare it against what steps 2 and 3 recorded
```

Step 2 draws objections. The current prompts hold **everything the feature learned** in a year, through edits nobody documented. **Rewriting them** loses that, along with the baseline that would show what the rewrite broke.

Step 3 draws the opposite mistake. **Deleting the stale eval set** removes a weak measurement and **replaces it with nothing**. A stale set is evidence about launch day; **rebuilding it from current traffic** makes it evidence about now.

## How the Exam Builds Wrong Answers

Wrong answers here describe things responsible teams really do: upgrade schedules, setup guides, notifying downstream owners. Each is wrong because of one assumption. Four cover almost everything.

- **A routine instead of a record.** The option gives a **process**, not a **value**: a setup procedure, a monthly upgrade, an agreement that authors will announce breakage, a hand-kept change log. Each depends on people doing it right every time, and none produces something two states can be compared against. Ask what you could **look up six weeks later**, and where.
- **A control removed to silence the symptom.** Unpinning so environments agree, deleting the stale eval set, dropping the component that broke. Each ends the complaint and removes what would make the **next** incident traceable. Ask whether the option leaves you **fewer declared variables**.
- **Communication instead of assessment.** Notifying owners, booking a window, staging the rollout, deferring the consumers nobody checked. All are competent handling of a change whose **effect nobody has measured**. Ask whether the option tells you anything new about **what** the change does, or only **when**.
- **Improvement before a baseline.** Upgrading the model, rewriting the prompts, adding a feature, on a system nobody has measured. They look like initiative. Ask what the result would be **compared against**.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| Something regressed and several things changed since the last good state | Explicit, pinned versions so the change can be traced and reversed | Upgrading more often, or removing the component that broke |
| Configuration the team needs lives on one person's machine | Move it to the project file that travels with the repo | A manual setup step, or a copy in each personal file |
| A personal experiment sits next to a shared requirement | Split by reach: shared to the project, experiment to the user | Merging them, or syncing personal files between machines |
| Prompts were edited in place and the eval set is older than current traffic | Pin inputs and record current behaviour before changing anything | Upgrading first, rewriting prompts, or deleting the old set |
| Each environment runs a different model, prompt and plugin set | One tracked config that declares the differences | Making environments identical, or unpinning so they agree |
| A prompt change has several downstream consumers | Assess the effect on each, then coordinate | Announcing the change, or trusting a clean staging run |
| A dependency changed overnight and nothing in the repo moved | A declared version range resolved from a tag | Trusting release notes to arrive before the breakage |
| Instruction and settings files are being set up for a new project | Version them with the code, reviewed the same way | Copies per environment, or a separate repository |
| Code uses a short model name and output changed with no deploy | Pin the canonical model ID and keep the previous one | Adjusting the parser to the new output |

One habit works beyond the table: for each option, ask what you could **read back six weeks from now**, and where. An option that only asks people to be careful leaves you **nothing to read**.

## What to Remember

- The **unattributable window** holds every **unrecorded change** since the last reproducible state. Configuration management keeps it **empty**.
- **Five things move silently**: the **model**, the **prompt**, **plugins**, **team config**, and the **eval set** traffic drifts away from.
- **Every Claude model ID is pinned.** From **4.6 on, the dateless ID is canonical** and never updated in place. **Older models' short aliases can move**; use the dated ID.
- The **model ID** and the **`anthropic-version`** header are **separate pins**.
- **Pinning narrows but doesn't remove change**: serving infrastructure can still shift behaviour slightly. Keep measuring.
- Every model ID has its **own retirement date**; **cloud partners set their own**. **Keep the previous ID** so a regression is a rollback, not a hotfix.
- A **prompt version** gives incidents something to **name** and rollbacks a **destination**. **Promote on a score**, not on reading the text.
- A prompt with **several consumers**: **assess each**. Announcing or staging isn't assessment.
- **Choose files by reach**: committed `CLAUDE.md`, `.claude/settings.json` and `.mcp.json` reach the team; `CLAUDE.local.md`, `.claude/settings.local.json` and `~/.claude.json` reach one person.
- **Precedence:** managed → command line → project local → shared project → user. **List keys merge.**
- **Instruction files are context**, not enforcement.
- **Plugin dependencies track latest by default.** Constrain them in `plugin.json`; tags are `{plugin-name}--v{version}`; **ranges intersect**. An upgrade schedule **isn't a record**.
- **Environments may differ on purpose**; **declare the differences** in one tracked file.
- On an untracked system, **restore measurability first**: pin, version prompts as they are, rebuild the eval set from current traffic, then change one thing.

## Endnotes for this chapter

1. Each Claude model ID identifies a pinned version, and the underlying model stays constant for the life of that ID; from the Claude 4.6 generation, dateless IDs are canonical and map to a single fixed snapshot whose weights and configuration Anthropic does not update, with updated versions shipping under new IDs; the belief that dateless IDs are evergreen pointers is called a common misconception. Models before 4.6 include a snapshot date, and on the Claude API also have shorter aliases pointing to the most recent dated snapshot for that minor version; the stability guarantee covers model IDs, not those aliases. Every model ID has its own deprecation and retirement schedule. Weights are fixed per ID, but serving infrastructure (router, safety classifiers, sampling logic) can change and occasionally produce minor behaviour differences. Anthropic, *Model IDs and versioning* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)

2. Retirement dates published by Anthropic apply to Anthropic-operated platforms; partner-operated platforms set their own schedules. Anthropic, *Model deprecations* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/about-claude/model-deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations)

3. Success criteria should be specific and measurable; code-based grading is fastest and most reliable but lacks nuance, LLM-based grading is flexible once its reliability is tested, and human grading is most flexible but slow and expensive. Anthropic, *Define success criteria and build evaluations* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/test-and-evaluate/develop-tests](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)

4. `.claude/settings.json` reaches teammates and cloud sessions only when committed; `.claude/settings.local.json` is added to the user's global git excludes when first written. When a key appears in several places, the highest level wins: managed settings, command line arguments, project local, shared project, user; list keys such as `permissions.allow` are combined across files. Anthropic, *Claude Code settings* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings)

5. CLAUDE.md files load into the context window at the start of every session and are treated as context, not enforced configuration; a project CLAUDE.md lives at `./CLAUDE.md` or `./.claude/CLAUDE.md` and is shared through version control; `CLAUDE.local.md` loads alongside it and should be git-ignored. Anthropic, *How Claude remembers your project* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory)

6. MCP server scopes: `local` (in `~/.claude.json` for the current project, only you), `project` (in `.mcp.json` at the project root, everyone who clones), and `user` (in `~/.claude.json`, only you, all projects). Anthropic, *Connect to MCP servers* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/mcp-quickstart](https://code.claude.com/docs/en/mcp-quickstart)

7. Managed settings are deployed as a `managed-settings.json` file in a system directory, as an MDM policy, or as server-managed settings from the claude.ai console. Anthropic, *Deploy managed settings* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/managed-settings](https://code.claude.com/docs/en/managed-settings)

8. A plugin dependency tracks the latest version by default; version constraints in the `dependencies` array of `.claude-plugin/plugin.json` hold it at a tested range; constraints resolve against git tags named `{plugin-name}--v{version}`; multiple constraints are intersected and resolved to the highest satisfying version, and auto-update stays within that range. Anthropic, *Constrain plugin dependency versions* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/plugin-dependencies](https://code.claude.com/docs/en/plugin-dependencies)

9. `enabledPlugins` turns individual plugins on or off, keyed by `plugin-name@marketplace-name`, in any settings file. Anthropic, *Claude Code settings reference* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/settings-reference](https://code.claude.com/docs/en/settings-reference)
