# Chapter 11: Four Mechanisms, Chosen by What Has to Run

**Summary**: *A capability arrives and four mechanisms are proposed for it. At least three of them would work, and that's the difficulty. Any rule that starts with "always prefer" has stopped reading the requirement, because the right choice depends on what's being packaged, not on a ranking of the packages. Two questions decide almost every case: does the work have to reach a live system, and how many consumers are there? This chapter covers those two questions, what each mechanism costs, and the loading rules that decide whether a package works on someone else's machine.*

## Four Proposals for One Capability, and None of Them Is Silly

The **month-end reconciliation report** has to come out exactly the way the finance team writes it: a **fixed section order**, **set wording**, and a checklist run before it goes anywhere.

Four proposals reach the meeting:

- Use a **built-in tool**, since Anthropic already ships one that runs code.
- Define a **custom tool** in the application, since every other capability lives there.
- Package the whole procedure as a **Skill**.
- Build an **MCP server**, since two other teams will want this by spring.

Each of those is real engineering, and someone shipped each of them this quarter. Ruling out the obviously silly one gets you nowhere, because all four survive a first read.

What separates them is **fit**, and fit is decided by two properties of the **capability**. Neither is a property of the mechanisms.

## The Execution Question Comes First, Because It Removes Two Options

Ask this before comparing anything: does the capability have to **reach a system** that holds **live data** and **authenticates its callers**, or is it **knowledge** about how work gets done?

**Procedure is knowledge.** A section order, a template, a house style, a review checklist, the three things people always forget. **None of it calls anything.** It's expertise written down, and the mechanism built for that is a **Skill**: a folder with a **`SKILL.md`** file (YAML frontmatter plus instructions) and any **templates and scripts** the procedure needs.<sup>[1]</sup>

**Live access is the other kind.** A balance that changed this morning, a record only certain callers may read, a write that has to land. **Text describing an endpoint can't call it**, and it drifts away from the real endpoint with nothing to notice.

The opposite mistake is just as tempting: **serving a procedure through an MCP server**. Putting a server in front of static instructions adds a network hop and no new ability.

One detail keeps the rule honest. A Skill **can include scripts**, and the runtime does run them.<sup>[1]</sup> But that doesn't give it **reach**. Through the API, Skills run inside the **code execution container**, which has **no network access** and **no package installation at runtime**.<sup>[1]</sup> So a bundled script does **local, predictable work**, not calls to production systems.

> **Example.** "Format the reconciliation report with these five sections and run this checklist" is a **Skill**. "Fetch this month's ledger balances from the finance system" needs to **execute** against a live system, so it's a tool or a server. Often you need both: a server to fetch the numbers, and a Skill describing how to present them.

## A Built-in Tool Is a Capability Anthropic Already Runs

Once something has to execute, three mechanisms remain. The cheapest definition is the one you never write, so check what **already exists** first.

"Built-in" means two different things, and the split matters:<sup>[2]</sup>

- **Server tools** such as web search, web fetch, code execution and tool search **run on Anthropic's infrastructure**. The call and its result both arrive in the response; **no code in your application** runs anything.
- **Anthropic-schema client tools** such as **bash and the text editor** come with a schema Anthropic publishes and trains Claude on, but **your application still runs** the operation and returns the result.

So a built-in saves you the definition, and sometimes the implementation. What it costs you is **ownership**. Chapter 9 put validation, approval and audit records in your dispatcher. For a server tool, there is **no dispatcher on a machine you control**. A built-in fits where the capability is **generic** and governance isn't the point. It's wrong the moment **someone has to answer** for what ran.

**Wrapping a built-in to make it shareable** fails in a specific way: the wrapper lives inside one application, so other teams can reach neither the wrapper nor the reason it exists.

## The Number of Consumers Decides Between a Custom Tool and a Server

Two mechanisms remain. They do the same thing, and what separates them isn't technical. **Count the applications.**

**One application**, a narrow operation, and an input contract you already know: that's a **custom tool**, defined where the application lives. In the **Agent SDK** it has **four parts**: a name, a description, an input schema, and the handler that runs. It's registered on an in-process server, so there's **no deployment, no transport choice, and nothing extra to version**.<sup>[3]</sup>

**Several applications**, on release schedules they don't share: that's an **MCP server** (chapter 10). A server is **machinery for reuse**, and you pay for it whether or not anyone reuses it: a process to deploy, a transport to choose, an interface to keep stable while callers change. Buy that for one consumer and you pay with nothing in return.

Run the logic the other way, and two comfortable answers fail:

- **Copying the implementation** into each team's app feels like independence but produces **divergence**. Every team owns a variant, and the next change to the underlying system happens in as many repositories as there are teams.
- **A shared library** looks better but moves the same cost one step: every consumer still integrates it, and every consumer must upgrade before a fix reaches anyone.

| What the capability needs | Who needs it | What fits | What you're signing up for |
|---|---|---|---|
| Nothing executed: procedure, templates, conventions | Anyone whose task matches the description | A Skill | A description precise enough to load, and a portable body |
| Generic execution Anthropic already runs | Whoever makes the request | A built-in tool | Governance you don't own, on work you don't see run |
| A private operation with a known input contract | One application | A custom tool | Nothing shared; a second consumer means doing it again |
| Live, authenticated access with its own release cycle | Several applications and teams | An MCP server | A deployment, a transport, and an interface that outlives its first caller |

Read the right-hand column as the price list. Most wrong answers here are **correct choices from the wrong row**, and the scenario always states which row it's in.

## A Skill Costs About a Hundred Tokens Until Someone Needs It

Why does a long procedure belong in a Skill, rather than in instructions every session carries? Because of **progressive disclosure**:<sup>[1]</sup>

| Stage | What's loaded | Cost |
|---|---|---|
| Always | The Skill's **name and description**, in the system prompt | About **100 tokens** per Skill |
| When a request matches the description | The **body of `SKILL.md`** | Under about **5,000 tokens** |
| When needed | **Reference files** (when read) and **scripts** (when run) | Nothing until used; scripts add **only their output** |

Because Claude never has to read a whole Skill to use part of it, what a Skill can carry is **effectively unlimited**.<sup>[4]</sup>

Compare that with the alternative. A procedure written into a **project instruction file** or system prompt is **paid for on every request**, including the many that have nothing to do with it. Claude Code's docs make the same point: a skill body **loads only when used**, so long reference material costs almost nothing until it's needed.<sup>[5]</sup>

That puts a lot of weight on one field. **The description isn't documentation; it's the matching rule.** It must say **what the Skill does and when to use it**.<sup>[1]</sup> A vague description doesn't fail gently. It **never loads**, on every surface, and the failure looks like a Skill that was never installed.

> **Example.** Weak: `description: Finance helper.` Strong: `description: Formats the month-end reconciliation report in the finance team's five-section layout and runs the pre-send checklist. Use when asked to write, review or finalise a month-end or reconciliation report.`

## One File, Several Runtimes, and Each Finds Skills Differently

The same `SKILL.md` runs in Claude Code, through the API, and under the Agent SDK. What changes is **how each runtime finds it** and **what it's allowed to touch**. "Runs everywhere" is something you design for.

- **Claude Code**: Skills are files. Personal ones go in **`~/.claude/skills/`**, project ones in **`.claude/skills/`**. When names clash, **enterprise overrides personal, and personal overrides project**.<sup>[5]</sup>
- **Agent SDK**: discovers the same filesystem locations through its setting sources, and a **`skills` option** lists which a session may use.<sup>[6]</sup>
- **Claude API**: custom Skills are **uploaded** through the Skills API, shared across the workspace, and referenced by `skill_id` and `version` together with the code execution tool. In production, **pin a version** so a Skill update can't quietly change behaviour.<sup>[7]</sup>
- **Managed Agents**: Skills are attached in the agent's `skills` array, or loaded from a mounted GitHub repository.<sup>[1]</sup>

**The file is portable; distribution isn't.** Custom Skills **don't sync between surfaces**. A Skill you want in three places is managed in three places.<sup>[1]</sup>

Then there's the failure everyone meets once: a packaged workflow **installs cleanly** on every team machine and **runs on only one**. Installing copies files; running resolves what those files point to. This book calls it the **author's-machine assumption**, and it's invisible to the author. It has three usual forms:

| What the author's machine supplied | Why installing still worked | What the package should have done |
|---|---|---|
| An absolute path into a home folder | A path is just text until something uses it | Use a path relative to the Skill, with forward slashes |
| A command available in the author's shell | Nothing in the package declares the dependency | Stick to what the runtime guarantees, or state the dependency |
| An environment variable set months ago | It's only read at the step that needs it | Name required packages and variables in the body |

Two more rules belong here:

- A Skill that uses MCP tools must give their **full names, including the server prefix** (`ServerName:tool_name`), or the lookup can miss when several servers are connected.<sup>[8]</sup>
- A **subagent starts clean**; it doesn't inherit Skills its parent loaded. List the Skills a subagent needs explicitly.

**Plugins** sit on top of all this. A plugin **bundles Skills, agents, hooks and MCP servers** into one versioned, installable unit, which is how a chosen set reaches a team.<sup>[9]</sup> It's **distribution, not a fifth answer**. A guardrail the author relied on locally only travels if the bundle actually contains it.

**Skills are code you run.** A Skill can direct Claude to call tools or run code that doesn't match its stated purpose, so treat a Skill from an unknown source like any other untrusted code, and review it before installing.<sup>[1]</sup>

## How the Exam Builds Wrong Answers

Wrong answers here name mechanisms that **work**. Each is someone's shipped architecture, which is why they sound reasonable. Four assumptions are worth memorising:

- **A universal preference stated as a rule.** "Prefer a Skill for anything procedural." "Standardise on servers." A correct statement says something true about the **work being packaged**; a wrong one crowns a mechanism regardless of the requirement. **Cost claims** are the usual disguise: "X is always cheaper" has priced nothing in the scenario.
- **Instructions where execution is needed.** API documentation, a procedure describing the query to run, a Skill standing in for a call. Ask what the option would **actually call**. A package that reaches nothing can't replace a capability that must reach something.
- **Reuse machinery for one consumer, or copies for many.** The same mistake in both directions. A private function doesn't need a deployment; four teams don't need four implementations; and a shared library keeps the coupling.
- **A recurring cost for an occasional need.** A long procedure loaded on **every request**, or a **nightly copy** of data standing in for live access. Both trade away what the requirement asked for (context or freshness) for convenience.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| A documented procedure with templates, conventions and a checklist, and nothing to call | A Skill, loaded when a request matches its description | A schema, which checks fields but carries no conventions |
| Several teams' applications need one live system, released independently | One MCP server, connected by each application | The same integration repeated per app, or endpoint documentation |
| One application, one narrow operation, a known input contract | A custom tool in that application | Server overhead for reuse that never happens |
| The capability is generic, and Anthropic already runs it | The built-in tool, as it is | A private reimplementation, or a wrapper nobody else can reach |
| Teams that don't share a codebase must each use a capability | One maintained artifact all of them call | Copies per team, or a library everyone must upgrade |
| The procedure shouldn't appear in unrelated requests | On-demand loading, paid for only by the description | Always-on instructions, paid for on every request |
| A packaged workflow installs everywhere but only works for its author | Remove environment assumptions from the package | Treating a clean install as proof the package works |

One habit works beyond the table. Before comparing options, answer the **two questions** from the scenario's own words: what has to **execute**, and how many **consumers** are named. An option that would be right under a different answer to either question is the one placed there to catch you.

## What to Remember

- The **capability decides** what fits, never a ranking of mechanisms. Any claim that one mechanism **always** wins, on cost or anything else, ignores the requirement.
- **First question: does it execute?** Reaching a **live, authenticated system** needs a tool or server. **Procedure** (templates, conventions, checklists) is what a **Skill** packages. Skill scripts run **sandboxed, with no network** on the API.
- **Second question: how many consumers?** **One application** with a known input contract → a **custom tool**. **Several applications** on independent release cycles → an **MCP server**, which you pay for whether or not reuse happens.
- **Built-ins** come in two kinds: **server tools** run on Anthropic's infrastructure with **no handler** in your app; **bash and the text editor** use Anthropic's schema but **run in your app**. You give up **governance at dispatch** for server tools.
- **Skills load in stages**: name and description always (**~100 tokens**), the body on a **description match** (**under ~5,000 tokens**), files and scripts only when used. That's why long procedures belong in Skills, not always-on instructions.
- The **description is the matching rule**. It says what the Skill does **and when to use it**. A vague one **never loads**.
- One `SKILL.md` runs everywhere, but **discovery differs**: folders in Claude Code and the SDK, **uploaded and version-pinned** on the API, a `skills` array in Managed Agents. Custom Skills **don't sync** between surfaces.
- A **clean install proves nothing**. Absolute paths, local commands and environment variables make a package work only for its author. **Subagents don't inherit Skills.** Use full `Server:tool` names for MCP tools.
- A **plugin** bundles Skills, agents, hooks and servers for **distribution**; it doesn't make the choice. Treat Skills from unknown sources as **untrusted code**.

## Endnotes for this chapter

1. Skills are filesystem-based resources that load on demand; each is a directory with a `SKILL.md` whose YAML frontmatter requires `name` and `description`, where the description states what the Skill does and when to use it; metadata loads at startup (about 100 tokens per Skill), the body loads when triggered (under 5k tokens), and bundled files cost nothing until accessed, with scripts contributing only their output; Skills on the API run in a sandboxed container with no network access and no runtime package installation; custom Skills don't sync across surfaces; in Managed Agents, skills are attached in the agent's `skills` array or loaded from a mounted GitHub repository; a malicious Skill can direct Claude to invoke tools or execute code that doesn't match its stated purpose. Anthropic, *Agent Skills* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)

2. Tools differ by where code runs: client tools, including user-defined tools and Anthropic-schema tools such as `bash` and `text_editor`, run in your application; server tools such as `web_search`, `web_fetch`, `code_execution` and `tool_search` run on Anthropic's infrastructure with no handler code in the application. Anthropic, *Tool use with Claude* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)

3. A custom tool has a name, a description, an input schema and an async handler; tools are registered on an SDK MCP server that runs in-process. Anthropic, *Give Claude custom tools* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/custom-tools](https://code.claude.com/docs/en/agent-sdk/custom-tools)

4. Skills complement MCP servers by teaching workflows that involve external tools, and because an agent with a filesystem doesn't need to read a whole Skill into context, the amount of context a Skill can bundle is effectively unbounded. Anthropic, *Equipping agents for the real world with Agent Skills* (Anthropic Engineering, accessed 2026-09-09; not re-checked for this edition), [https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

5. Unlike CLAUDE.md content, a skill body loads only when used; where a skill is stored decides who can use it, and name conflicts resolve with enterprise over personal over project. Anthropic, *Extend Claude with skills* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills)

6. In the Agent SDK, skills are `SKILL.md` files loaded from filesystem locations governed by setting sources, discovered at startup as metadata, and scoped with a `skills` option. Anthropic, *Extend agents with skills* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/skills](https://code.claude.com/docs/en/agent-sdk/skills)

7. On the Messages API, Skills are referenced by `skill_id` and `version` in the `container` parameter with the code execution tool; custom Skills are uploaded through the Skills API and shared workspace-wide; production guidance is to pin a specific version. Anthropic, *Using Agent Skills with the API* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/skills-guide](https://platform.claude.com/docs/en/build-with-claude/skills-guide)

8. Use forward slashes in paths, don't assume packages are installed, and use fully qualified MCP tool names in the form `ServerName:tool_name`. Anthropic, *Skill authoring best practices* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

9. Plugins bundle skills, agents, hooks and MCP servers into one installable unit, and plugin skills are namespaced with the plugin's name. Anthropic, *Create plugins* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/plugins](https://code.claude.com/docs/en/plugins)
