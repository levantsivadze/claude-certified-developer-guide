# Chapter 14: Permission Modes, and Where the Human Check Goes

**Summary**: *Claude Code is configured before it's trusted, and most configuration decisions are made once and then inherited by everyone afterwards. This chapter covers which permission mode fits the risk of the work, which file a setting belongs in so it reaches the people who need it, why a standard that applies everywhere is written once instead of copied into every repository, what each core component is for, and how an unattended run gets its output format enforced instead of requested. Set the scope wrong, and one afternoon's convenience becomes everyone's default.*

## The Mode Was Widened for One Refactor and Never Narrowed Again

A **rename across forty files** is painful one approval at a time. So someone **widens the permission mode**, finishes the rename, and moves on. That was right for the rename. But **nobody sets it back**. It ends up in the **project settings file**, gets committed, and a session opened to change one value now starts with the authority of a session meant to rewrite a whole package.

This book calls that a **standing grant**: authority given **once**, in a file, and then **inherited by every later session** without anyone asking whether the current task needs it. Permission modes, settings files, context files and allow rules are all standing grants. Most scenarios here are a standing grant meeting work it was never sized for.

## Plan Mode Separates Deciding What to Change From Changing It

Claude Code works through **three blended phases**: gather context, act, verify.<sup>[1]</sup> The recommended workflow **walls off the first from the second**. In **plan mode**, Claude reads, searches and explores **without touching your source**; you approve a plan; then the implementation runs.<sup>[2]</sup> That wall is real, not advisory, because plan mode is itself a permission mode, and **edits stay blocked until you approve**.<sup>[3]</sup>

That fits a common scenario. A behaviour is spread across **several entry points**, and earlier edits kept landing in the wrong one. More reading has no end; following test failures assumes the tests cover every entry point, which is exactly what's in doubt; one module per session guarantees nobody sees the whole behaviour. **Map the flows, compare the places to change, then edit.**

> **Example.** "Discounts are applied twice for some orders." In plan mode, ask Claude to find every place a discount is calculated (checkout, the cart API, the admin order editor, the nightly batch). It lists four call sites and proposes changing the one shared function they should all use. You approve that plan, and only then does it edit.

Gathering context rewards precision too. A set of files defined by **name** ("all `*.test.ts` files") is found exactly with a **path pattern** (glob). A **content search** returns whatever mentions a word, and misses files that never do.

## Six Permission Modes, Read by Who Decides What the Mode Doesn't Cover

A **mode sets your session's baseline**. Read the list by **who decides** everything the mode doesn't cover, not by how much it lets through:<sup>[3]</sup>

| Mode | Runs with no prompt | Who decides the rest | Best for |
|---|---|---|---|
| `default` (shown as "Ask before edits") | Reads | A person, at a prompt | Getting started, sensitive work |
| `acceptEdits` | Reads, file edits, common file commands like `mkdir`, `touch`, `mv`, `cp` | A person, at a prompt | Iterating on code you're reviewing |
| `plan` | Reads | A person; edits wait for an approved plan | Exploring before changing |
| `auto` | Everything, with background safety checks | A **classifier model**, reviewing each action before it runs | Long tasks, reducing prompt fatigue |
| `dontAsk` | Only pre-approved tools | **Nobody**: anything that would prompt is **denied** | Locked-down CI and scripts |
| `bypassPermissions` | Everything | **Nobody** | **Isolated containers and VMs only** |

Two facts hold across every row:

- **Deny rules and explicit ask rules apply in every mode**, including `bypassPermissions`. **Allow rules mean nothing** in `bypassPermissions`, because everything is already approved.<sup>[3]</sup>
- Writes to **protected paths** such as `.git` and `.claude` are never auto-approved outside `bypassPermissions`.<sup>[3]</sup>

You switch modes with **Shift+Tab** (cycling `default → acceptEdits → plan`, plus any optional modes you've enabled), with `--permission-mode` at startup, or with `permissions.defaultMode` in settings. `dontAsk` never appears in the cycle; set it with the flag.<sup>[3]</sup> Recent versions **ignore `defaultMode: "auto"` in project settings files**, so a cloned repository can't switch you into auto mode; set it in user or managed settings.<sup>[3]</sup>

The mistake is **widening a mode to stop the prompts**. Prompt fatigue is a real cost, but it isn't a risk assessment. **Size the mode to the blast radius.** A deny rule at project or organisation scope closes what a mode leaves open.

> **Example.** A project's `.claude/settings.json`:
> ```json
> {
>   "permissions": {
>     "allow": ["Bash(npm run test:*)", "Bash(npm run lint)"],
>     "deny": ["Bash(rm -rf:*)", "Read(./.env)", "Bash(git push:*)"]
>   }
> }
> ```
> Tests and lint run without prompts; deleting trees, reading secrets and pushing are blocked **in every mode**.

## Configuration Scope Follows Who Needs to Receive the Setting

Claude Code reads settings from **four files**, plus whatever an organisation deploys. The file a key is in decides who gets it. **Precedence**, highest first:<sup>[4]</sup>

1. **Managed settings** (deployed by the organisation; nothing overrides them)
2. **Command-line flags**
3. **`.claude/settings.local.json`** (personal, in one project, not committed)
4. **`.claude/settings.json`** (committed, shared with the team)
5. **`~/.claude/settings.json`** (personal, every project on your machine)

**List settings** like `permissions.allow` **merge across files** instead of replacing each other.<sup>[4]</sup>

Read that stack as a question about **audience**. A setting everyone on the project needs goes in the **committed project file**: it travels with the checkout, gets reviewed like the code it governs, and your teammate's clone has it. The user file covers every project on **your** machine and reaches nobody else. The local file is a **personal exception** inside one project.

Two named failures follow:

- **Environment variables carrying a team decision** leave the value on one machine, unreviewed and unversioned.
- **Documentation offered as configuration** (a wiki page, a preamble everyone pastes) depends on people repeating themselves exactly, and nothing loads it.

## Context Files Stack Instead of Overriding, So a Standard Is Written Once

**`CLAUDE.md`** is the persistent instruction file, **loaded at the start of every session**.<sup>[5]</sup> It lives at project scope in `./CLAUDE.md` or `./.claude/CLAUDE.md`, at user scope in `~/.claude/CLAUDE.md`, and as a managed policy file.<sup>[6]</sup> Run **`/init`** and Claude drafts a starter from the repository.<sup>[7]</sup>

The key fact: every discovered file is **concatenated, not overridden**, ordered from the **broadest scope down** to your working directory.<sup>[5]</sup> Nothing replaces anything. So you write a standard that must hold everywhere **once, at the wide scope**, and each repository's file carries only what's true for that repository. Copy the block into every repository instead, and you get as many copies as repositories, drifting apart.

Two limits apply:

- **Long files are followed less well.** Keep `CLAUDE.md` **under about 200 lines**, and move narrower guidance into **`.claude/rules/`**, where a **`paths` glob** loads a rule only when Claude reads a matching file.<sup>[5]</sup>
- **`CLAUDE.md` is context, not enforcement.** To block an action no matter what Claude decides, use a **hook** (chapters 13 and 21).<sup>[5]</sup>

> **Example.** `.claude/rules/migrations.md`:
> ```markdown
> ---
> paths: ["db/migrations/**"]
> ---
> Never edit an existing migration. Create a new one with `make migration name=...`.
> ```
> This rule only enters context when Claude reads a file under `db/migrations/`, so it costs nothing in sessions that don't touch migrations.

## Five Components, Separated by When They Load and Who Starts Them

| Component | Where it lives | When it enters context |
|---|---|---|
| **Rules** | `.claude/rules/*.md` | Every session, or only when a `paths` glob matches a file being read |
| **Skills** | `.claude/skills/<name>/SKILL.md`, or under `~/.claude` | Description at startup; the body when Claude judges it relevant or someone types `/name` |
| **Commands** | `.claude/commands/*.md` | When someone types `/name` |
| **Subagents** | `.claude/agents/*.md`, or under `~/.claude` | In their own context window, when work is delegated |
| **Auto memory** | `~/.claude/projects/<project>/memory/` | The `MEMORY.md` index every session (first **200 lines or 25KB**); topic files on demand |

A **skill is a folder**, and that decides a recurring scenario.<sup>[8]</sup> A procedure made of **instructions plus helper scripts** has somewhere to keep both, and its body **costs no context** until used. The same material in a context file loads every session whether it's needed or not, and the scripts have nowhere to live. Built as a library, it becomes something application code calls, when the requirement was that a **session** could use it. Command files still work, but **skills are their successor**.<sup>[6]</sup>

**Subagents** run in their own context window with their own tool access, so a long exploration **returns a summary** and keeps the main conversation clean.<sup>[9]</sup> **Auto memory** goes the other way: Claude writes it for itself, and because it's **local to one machine**, it shares nothing with the team.<sup>[5]</sup>

## An Unattended Run Needs Its Output Format Enforced, Not Requested

**`claude -p "..."`** runs one prompt and exits (formerly called **headless mode**).<sup>[6]</sup> **`--output-format`** chooses `text`, `json`, or `stream-json` (newline-delimited events; with `--verbose` and `--include-partial-messages`, it streams tokens as they're generated). The decisive flag is **`--json-schema`**: together with JSON output, it returns the result in a **`structured_output`** field that matches your schema, and an invalid schema is rejected at startup.<sup>[10]</sup>

That settles a family of automation scenarios. A pipeline that turns a review into inline PR comments needs **one finding per object**, with fields it can index. A firmer instruction still only asks. A regex over markdown turns this week's formatting into a contract. A "cleanup pass" runs a second unconstrained generation over the first. **Parsing prose instead of using a contract** is the mistake.

> **Example.**
> ```bash
> claude -p "Review the diff in pr.diff for bugs" \
>   --output-format json \
>   --json-schema '{"type":"object","properties":{"findings":{"type":"array","items":{"type":"object","properties":{"file":{"type":"string"},"line":{"type":"integer"},"severity":{"enum":["low","medium","high"]},"message":{"type":"string"}},"required":["file","line","severity","message"]}}},"required":["findings"]}' \
>   --permission-mode dontAsk --allowedTools "Read" \
>   | jq '.structured_output.findings'
> ```

Two more details:

- A **`-p` run starts in `default` mode**, so give an unattended job its mode explicitly. **`dontAsk`** denies anything outside the pre-approved set instead of waiting for input nobody will give.<sup>[3]</sup>
- **`--bare`** skips auto-discovery of hooks, skills, commands, subagents, plugins, MCP servers, memory and `CLAUDE.md`, so a run behaves the **same on a machine nobody configured**.<sup>[10]</sup>

## Put the Human Check Where Mistakes Are Expensive, Not Where Habit Puts It

Modes and rules settle what runs without asking. They don't say where a person still needs to look. That decision has one input: **what a wrong call costs, and how hard it is to undo**.

**Sort actions by what undoes them:**

| How it's undone | Example | What it needs |
|---|---|---|
| A checkpoint or a diff | Editing files in the working directory | No gate; review the diff |
| Only by another action | A destructive shell command, a write outside the project | A pause by default, plus a **deny rule** to make it reliable |
| Nothing can undo it | Sending an email, a production deploy, deleting a database | **A person must approve.** Claude's own review is an input, not a substitute |

A mode and a gate are **one judgement seen twice**. A mode sets a session's default posture; a gate lifts one action out of it.

## How the Exam Builds Wrong Answers

Wrong options here describe configurations real teams ship. What makes one wrong is the assumption underneath, and four cover almost everything:

- **Convenience treated as a requirement.** A mode is picked because it removes interruptions, and the **risk of the work never enters** the comparison. It's correct only where the scenario says the work is **low-stakes and reversible**.
- **State that doesn't travel, offered as shared configuration.** Local environment variables, a pasted preamble, a wiki page. Each holds the content; none arrives with a checkout or loads in a session.
- **Copying instead of using a scope that already exists.** A shared standard copied into every repository, or linked from each. **Layering already combines** general and local instructions.
- **A format requested where it had to be enforced.** Stronger wording, cleverer parsing, or a second generation to tidy the first. All leave the consumer depending on formatting nothing promised.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| Configuration must be identical for everyone on a repository | The committed project settings file | Per-machine environment variables, or a document to follow |
| Standards apply everywhere, with extra rules in some repositories | Write the general standard once at the wide scope; keep local files local | Copying the block into each repository, or linking to a page |
| A program consumes the run's output | `--json-schema` (or structured outputs), then validate on receipt | A firmer instruction, a regex over markdown, or a cleanup pass |
| Edits keep landing in the wrong place in unfamiliar code | Plan mode: map the flows and compare change points before editing | More reading, test-driven guessing, or one narrow session per module |
| A team procedure includes instructions plus helper scripts | A skill folder the tool discovers and loads on demand | Putting it in `CLAUDE.md`, or shipping it as a library |
| The action is hard to undo or reaches outside the repository | A pause for a person, backed by a deny rule | A mode left over from the last task |
| A job runs unattended and must behave the same on every machine | Pre-approved tools, `dontAsk`, `--bare`, and a fixed output schema | A default that assumes someone is watching |

One habit works beyond the table. Before comparing options, ask **what a fresh clone on an unconfigured machine would inherit**.

## What to Remember

- A **standing grant** is authority **set once in a file** and **inherited by every later session**. Modes, settings files, context files and allow rules all qualify.
- **Six modes**: `default` (reads only), `acceptEdits` (edits and common file commands), `plan` (**edits blocked** until a plan is approved), `auto` (a **classifier** reviews each action), `dontAsk` (**pre-approved tools only, never waits**), `bypassPermissions` (**isolated containers only**). **Deny and ask rules hold in every mode**; allow rules mean nothing in `bypassPermissions`.
- **Settings precedence**, highest first: **managed**, **CLI flags**, **`settings.local.json`**, **committed `settings.json`**, **user settings**. List keys like `permissions.allow` **merge**.
- **Scope follows audience.** Team-wide rules go in the **committed project file**. Environment variables and wiki pages reach nobody.
- **`CLAUDE.md` files stack**, broadest first; nothing overrides. Write a shared standard **once at the wide scope**. Keep each file **under ~200 lines**; use **`.claude/rules/` with `paths`** for narrow guidance.
- **`CLAUDE.md` is context, not enforcement.** Absolute prohibitions belong in **hooks** or **deny rules**.
- **Five components by load behaviour**: rules (every session or on a path match), skills (body on demand; folders that can hold scripts), commands (when typed), subagents (own context and tools), auto memory (written by Claude, **local to one machine**).
- **`-p`** runs once and exits; **`--json-schema`** returns a validated **`structured_output`**. Unattended jobs **enforce** the format. A `-p` run starts in `default`; set the mode explicitly. **`--bare`** skips all local discovery.
- **Place the human check by reversibility**: none where a diff undoes it, a pause plus a deny rule where only another action can, and **a person** where nothing can.

## Endnotes for this chapter

1. Claude works through three blended phases: gathering context, taking action and verifying results. Anthropic, *How Claude Code works* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works)

2. The recommended workflow is to explore in plan mode, ask for a detailed plan, implement after approval, and commit; planning helps most when the approach is uncertain, the change spans several files, or the code is unfamiliar. Anthropic, *Best practices for Claude Code* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)

3. The permission modes and what each runs without a prompt: `default` reads only; `acceptEdits` reads, file edits and common filesystem commands such as `mkdir`, `touch`, `mv` and `cp`; `plan` reads only; `auto` everything, with background safety checks; `dontAsk` only pre-approved tools, for locked-down CI and scripts; `bypassPermissions` everything, for isolated containers and VMs only. Deny rules and explicit ask rules apply in every mode including `bypassPermissions`, where allow rules have no effect; Shift+Tab cycles `default`, `acceptEdits` and `plan`, with optional modes after; `dontAsk` never appears in the cycle; `defaultMode` sets the starting mode; recent versions ignore `defaultMode: "auto"` from `.claude/settings.json` and `.claude/settings.local.json`. Anthropic, *Choose a permission mode* (Claude Code Docs, accessed 2026-11-27), [https://code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes)

4. The settings files and their scope; precedence from managed settings through command-line arguments, local project, shared project and user settings; list keys such as `permissions.allow` merge across files. Anthropic, *Claude Code settings* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings)

5. CLAUDE.md loads at the start of every session and is context, not enforced configuration, with a `PreToolUse` hook named for blocking actions regardless of Claude's decision; discovered files are concatenated from the filesystem root down to the working directory; target under 200 lines per file; `.claude/rules/` files can be scoped with a `paths` glob; auto memory is per repository under `~/.claude/projects/`, machine-local, with the first 200 lines or 25KB of `MEMORY.md` loaded every conversation. Anthropic, *How Claude remembers your project* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory)

6. Glossary definitions: CLAUDE.md scopes; rules in `.claude/rules/`; commands in `.claude/commands/`, with skills as the recommended successor; skills loaded when relevant or invoked directly; subagents with their own context and tool access; non-interactive mode with `-p`, formerly headless mode; bare mode skipping hooks, skills, commands, subagents, plugins, MCP servers, auto memory and CLAUDE.md. Anthropic, *Glossary* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/glossary](https://code.claude.com/docs/en/glossary)

7. `/init` generates a CLAUDE.md for a project. Anthropic, *Commands* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands)

8. A skill is a directory with `SKILL.md` and optional supporting files and scripts, discovered from `.claude/skills/` and `~/.claude/skills/`, with the body loading only when used. Anthropic, *Extend Claude with skills* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills)

9. Subagents keep exploration out of the main conversation, can be limited to specific tools, and are defined in `.claude/agents/` or `~/.claude/agents/`. Anthropic, *Create custom subagents* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents)

10. `-p` runs non-interactively; `--output-format` accepts `text`, `json` and `stream-json`, the last used with `--verbose` and `--include-partial-messages` for streaming; `--output-format json` with `--json-schema` returns a `structured_output` field, and an invalid schema is rejected at startup; `--bare` skips auto-discovery so a run behaves the same on every machine. Anthropic, *Run Claude Code programmatically* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/headless](https://code.claude.com/docs/en/headless)
