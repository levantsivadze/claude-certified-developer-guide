# Chapter 10: One Server, Many Clients, Transport Chosen by Location

**Summary**: *One capability, four teams, and a vendor API that changes whenever it likes. Wire that integration into each application, and the same fix happens four times, late, in four codebases. An MCP server is the alternative: one implementation, several callers, released on its own schedule. This chapter covers what a server exposes and who decides to use each part, why the transport depends on where the server runs, which configuration scope shares a server and which keeps it private, what changed in the 2026 version of the protocol, and what changes when callers have different privileges.*

## Four Applications, One Interface, Four Copies of Every Breaking Change

Picture an internal **analytics platform** with **four applications** that want to use it: a support assistant, a reporting agent, a developer tool, and something an outside contractor uses. Each team writes its own schema, authentication and error handling. That's **four descriptions of one operation**.

Then the platform ships a **breaking change**, on a schedule none of the four teams controls. That's **four fixes**, found at four different times.

That's the case for a server, and it's about **reuse, not capability**. The **Model Context Protocol (MCP)** takes the tool definition out of any single application and puts it in a **separate process** that clients connect to.<sup>[1]</sup> You build the operation once, and every connected client gets it.

This book calls that line the **reuse boundary**: the point where a capability stops belonging to one application and becomes a separately released piece several applications call. Two facts push a capability over it:

- **More than one consumer.**
- An interface that **changes independently** of all of them.

With one consumer and a stable interface, a tool wired directly into the application **stays cheaper**. Chapter 11 covers that choice.

## Three Building Blocks, Separated by Who Decides to Use Them

A server exposes **tools, resources and prompts**. Descriptions usually explain what each one contains, which is the least useful thing about them. What matters is **who starts the use**.

| Building block | Who starts it | What it holds | Use it when |
|---|---|---|---|
| Tool | **Claude**, based on the request | An action with a schema, called and returning a result | The operation must be chosen at runtime |
| Resource | **The host application**, which assembles context | Read-only data with a URI | Known data belongs in context from the start of the turn |
| Prompt | **A person**, choosing it on purpose | A vetted instruction template, usually shown as a slash command | Exact wording matters more than whatever someone would type |

Tools are **model-controlled**.<sup>[2]</sup> Resources are **application-driven**: the host decides how to bring them in.<sup>[3]</sup> Prompts are **user-controlled**: someone picks one explicitly.<sup>[4]</sup>

Each wrong choice has a cost:

- **Data published as a tool** wastes a round trip fetching something that was never in doubt.
- **Wording published as a tool** leaves Claude deciding whether to use the exact phrasing that was the whole point.

> **Example.** A code-review server might offer a **tool** `get_pull_request(number)` (Claude decides which PR to read), a **resource** `repo://style-guide` (the app loads the style guide into every review), and a **prompt** `/security-review` (a reviewer picks it when they want a security-focused pass with vetted wording).

## The Transport Depends on Where the Server Runs

MCP defines two standard **transports**:<sup>[5]</sup>

- **stdio**: the client starts the server as a **child process**, and they talk over its standard input and output, **one JSON-RPC message per line**. Standard error is free for logging; standard output may contain **only valid messages**.
- **Streamable HTTP**: the server exposes **one HTTP endpoint**.

So **where the server runs decides**. A child process already has its pipes. A deployment **shared by many users**, on a machine none of them own, is what a network protocol is for.

Two mistakes follow, and both sound like consistency:

- **A listening port on every workstation** gives a purely local tool a network attack surface nobody needed.
- **A subprocess per user of a shared deployment** multiplies server processes with users, and gives up the single deployment that was the reason to build a server.

**Transport is chosen per deployment**, not fixed by the protocol. One server can run over pipes locally and be hosted over HTTP for the shared case. A third option skips the process boundary entirely: an **SDK MCP server** defined in your own application code, running in-process.<sup>[6]</sup>

Claude Code's configuration follows the same idea. An entry with a **`command`** is stdio by default. An entry with a **`url`** needs an explicit `type` of **`http`** (or its alias `streamable-http`), `sse` or `ws`. A `url` with no `type` is read as stdio, skipped, and reported as an error.<sup>[7]</sup> The older **HTTP+SSE** transport is formally **deprecated** in the protocol; migrate SSE servers to Streamable HTTP.<sup>[13]</sup>

## The 2026 Protocol Is Stateless

The **MCP 2026-07-28** specification, released in July 2026, is the biggest revision since launch, and it includes **breaking changes**.<sup>[13]</sup><sup>[14]</sup>

- **No more sessions.** Before, a connection started with an `initialize` handshake, and HTTP servers could keep session state. Now each request **carries its own protocol version, client identity and capabilities** in `_meta`. Any request can land on any server instance behind an ordinary load balancer, with no shared storage.
- **Servers must implement `server/discover`**, which advertises their supported versions, capabilities and identity.
- **Every result has a `resultType`**: `"complete"`, or `"input_required"` for **multi round-trip requests**, the new way a server asks the client for something in the middle of a call.
- **`tools/list` results can be cached** for as long as the server allows.
- **Roots, Sampling and Logging are deprecated.** The documented replacements are tool parameters or resource URIs instead of Roots, calling an LLM provider's API directly instead of Sampling, and stderr or OpenTelemetry instead of Logging.
- A **formal deprecation policy** promises at least **twelve months** between deprecation and removal.

> **Example.** Under the old protocol, a remote server running on three instances needed "sticky" load balancing or a shared session store, because the session lived on one instance. Under 2026-07-28, a plain round-robin load balancer works, because each request carries everything the server needs.

Anthropic is rolling out 2026-07-28 support across Claude products.<sup>[15]</sup>

## The Host Owns the Client, and a Server Never Sees the Whole Conversation

The architecture is **host, client, server**. The host application creates and manages clients, **each client talks to exactly one server**, and the host enforces the boundaries between them.<sup>[1]</sup>

That has three consequences for anyone writing a server. A server receives **only what's sent to it**, **can't read the conversation history**, and **can't see into another server**. So nothing it does can depend on context it wasn't given. That isolation is also what lets several servers combine safely.

You **write the server**. You **rarely write the client**: Claude Code includes one, and the **MCP connector** reaches a remote server straight from the Messages API, with no separate client process.<sup>[8]</sup>

## Configuration Scope Decides Who Loads the Server, Nothing Else

Claude Code saves a server definition at one of **three scopes**, plus a fourth set by administrators:<sup>[9]</sup>

| Scope | Where it's saved | Who gets the server |
|---|---|---|
| `local` (the default) | `~/.claude.json`, under this project | One person, one project |
| `project` | `.mcp.json` at the repository root | Everyone who clones the repository |
| `user` | `~/.claude.json`, at the top level | One person, every project |
| Managed | A file deployed by administrators | Everyone in the organisation, with no setup |

**Scope follows who needs the server.** A dependency the team shares belongs in **version control**, reviewed and versioned with the code. A personal experiment stays personal. **Setup steps in a README** are the mistake that keeps winning arguments it should lose: they can't be reviewed, versioned or distributed, and they drift the first time someone reads step four differently.

> **Example.** Your team's issue tracker server is added with `claude mcp add --scope project tracker -- npx tracker-mcp`. That writes `.mcp.json`, which you commit. Everyone who clones the repo gets the same server, after approving it once.

Four details matter:

- **Scope is fixed** when you add a server. To change it, remove the entry and add it again.<sup>[10]</sup>
- If one name is defined at several scopes, the **highest-precedence definition is used whole**. Fields aren't merged across scopes.<sup>[9]</sup>
- The first time Claude Code sees a project-scoped server, it **asks for approval**, so a cloned repository can't start a process without your consent.<sup>[10]</sup>
- A project-scoped stdio server runs on **each teammate's machine**, so every clone needs the runtime installed. Never put a secret inline in `.mcp.json`: **commit history keeps it** long after you remove it. Chapter 22 covers credentials.

## A Server Used by Callers With Different Privileges Is a Trust Boundary

Once a server is truly shared, your callers **stop being alike**. One of those four teams is contractors. Three decisions follow, and each has a comfortable alternative that only works while **every caller is honest**.

**Authenticate and authorise at the server.** Work out who the caller is, instead of treating every connected client as equally trusted. An **identity the caller supplies is a claim**, not a verification. The spec puts the credential rule as a prohibition: a server **must not accept a token that wasn't issued for it**. A server that forwards unchecked tokens is a proxy an attacker can point at whatever sits behind it.<sup>[11]</sup>

**Choose tool size deliberately.** A small set of **purpose-specific operations** can each be authorised per caller. A thin pass-through to every endpoint can't, and neither can one tool that accepts any query. That last one looks flexible, but it's **blast radius**: a tool that can express any statement gives you nothing to authorise separately.

> **Example.** Instead of `run_sql(query)`, expose `get_weekly_revenue(region)` and `list_top_customers(limit)`. Now the contractor's client can be allowed the first and denied the second, and no caller can run `DROP TABLE`.

**Return errors as structured results** Claude can reason about, with **no stack traces** and **no connection strings**. Chapter 9 made the general case for useful errors. What's new here is that the person reading this one is **outside the team** that wrote it.

**Nobody audits this for you.** Anthropic reviews servers before listing them in its directory but **doesn't security-audit** them. An organisation that needs a fixed set deploys one through managed configuration, or uses allowlists and denylists.<sup>[12]</sup>

## How the Exam Builds Wrong Answers

Wrong answers here describe real engineering: standardising on one transport, scoping a server to the team that asked first, keeping a configuration personal. Four assumptions make them wrong:

- **One mechanism said to win everywhere.** "Prefer the network transport", "run it locally", "support both". A universal preference ignores the **deployment facts** in the scenario.
- **A design that only works while callers are honest.** Accepting a supplied identity, a general query tool, internal details in errors. The clue is any stated **difference in privilege** between consumers.
- **An interface built for the first consumer.** Building against one application and extending later, or deploying only to developer machines. Both bring back the coupling a server exists to remove.
- **Scope chosen for convenience, not audience.** Personal configuration for a shared dependency, or manual steps instead of a committed file.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| Several applications need one capability, released separately | One server with an interface independent of any consumer, hosted where all can reach it | Per-application integration, or an interface built for the first consumer |
| A server runs as a local child process on one machine | stdio | A network transport, a listening port, or both |
| One server serves workstations and a shared hosted application | A transport per deployment: stdio locally, Streamable HTTP for the shared case | One transport forced on both, or a subprocess per user |
| A remote server must scale across many instances | Streamable HTTP under the stateless 2026-07-28 protocol, behind an ordinary load balancer | Sticky sessions, or the deprecated HTTP+SSE transport |
| Data is known ahead of time and needed at the start of the turn | A resource, loaded into context | A tool call to fetch something never in doubt |
| Every teammate needs the same server after cloning | Project scope, committed in `.mcp.json` | Personal scope, or setup steps in a README |
| Some consumers are outside the team | Authenticate at the server, scope tools narrowly, keep internals out of errors | Trusting the connection, or a general query tool called "flexible" |

One habit works beyond the table. Name the **deployment facts** the scenario gives: how many consumers, where each runs, and how their privileges differ. Then check whether the option uses them.

## What to Remember

- A **server earns its place** with **more than one consumer** and an **interface that changes independently** of them. One consumer and a stable interface make a directly wired tool cheaper.
- **Tools, resources and prompts** differ by **who starts them**: **Claude calls a tool**, the **host loads a resource** (read-only data with a URI), and a **person picks a prompt** (a vetted template, usually a slash command).
- **Where the server runs decides the transport.** A **child process** uses **stdio**, one JSON-RPC message per line. A **shared deployment** uses **Streamable HTTP**. Choose **per deployment**. HTTP+SSE is **deprecated**.
- **MCP 2026-07-28 is stateless**: no sessions, protocol version and capabilities **sent with every request**, `server/discover`, cacheable `tools/list`, multi round-trip requests, and **Roots, Sampling and Logging deprecated**.
- The **host creates one client per server**. A server **never sees the conversation history** or other servers.
- **Scope decides who loads a server**: **`local`** (one project, private, the default), **`project`** (committed `.mcp.json`, everyone who clones), **`user`** (private, every project), and **managed** (organisation-wide). Scope is **fixed** when added; definitions **aren't merged** across scopes; project servers need **one-time approval**.
- A **shared server is a trust boundary**. **Verify identity** at the server, since a supplied identity is only a claim, and **never accept tokens issued for something else**. **Scope tools to named operations**, because a tool that accepts any query gives you nothing to authorise.

## Endnotes for this chapter

1. MCP follows a client-host-server architecture: the host creates and manages clients, each client talks to exactly one server, and the host enforces security boundaries; servers expose resources, tools and prompts and can't read the whole conversation or see into other servers. Model Context Protocol, *Architecture* (accessed 2026-09-09; not re-checked for this edition), [https://modelcontextprotocol.io/specification/draft/architecture](https://modelcontextprotocol.io/specification/draft/architecture)

2. Tools are model-controlled: the language model discovers and invokes them based on context and the user's prompts. Model Context Protocol, *Tools* (accessed 2026-09-09; not re-checked for this edition), [https://modelcontextprotocol.io/specification/draft/server/tools](https://modelcontextprotocol.io/specification/draft/server/tools)

3. Resources share data such as files or database schemas, each identified by a URI; they are application-driven, with the host deciding how to include them. Model Context Protocol, *Resources* (accessed 2026-09-09; not re-checked for this edition), [https://modelcontextprotocol.io/specification/draft/server/resources](https://modelcontextprotocol.io/specification/draft/server/resources)

4. Prompts are user-controlled templates, exposed so the user selects them explicitly, typically as slash commands. Model Context Protocol, *Prompts* (accessed 2026-09-09; not re-checked for this edition), [https://modelcontextprotocol.io/specification/draft/server/prompts](https://modelcontextprotocol.io/specification/draft/server/prompts)

5. The standard transports are stdio, where the client launches the server as a subprocess exchanging newline-delimited JSON-RPC messages over standard input and output, with standard error for logging and nothing but valid messages on standard output; and Streamable HTTP, with a single HTTP endpoint. Model Context Protocol, *Transports* (accessed 2026-09-09; not re-checked for this edition), [https://modelcontextprotocol.io/specification/latest/basic/transports](https://modelcontextprotocol.io/specification/latest/basic/transports)

6. In the Agent SDK, a server run by a command uses stdio, a server at a URL uses HTTP, and tools built in the application's own code use an in-process SDK MCP server. Anthropic, *Connect to external tools with MCP* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/mcp](https://code.claude.com/docs/en/agent-sdk/mcp)

7. A JSON entry with a `url` but no `type` is read as stdio, skipped, and reported; `type` must be `http`, `sse` or `ws`, and `streamable-http` is accepted as an alias for `http`. Anthropic, *Connect Claude Code to tools via MCP* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp)

8. The MCP connector connects to remote MCP servers directly from the Messages API without a separate client. Anthropic, *MCP connector* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/agents-and-tools/mcp-connector](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector)

9. Local scope is the default, stored in `~/.claude.json` under the project and private; project scope is a committed `.mcp.json` at the repository root; user scope is in `~/.claude.json` across all projects; administrators can deploy managed configuration. The highest-precedence definition of a name is used whole, without merging fields across scopes. Anthropic, *Connect Claude Code to tools via MCP* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp)

10. A server's scope is fixed when added, so changing it means removing and re-adding the entry; the first time Claude Code sees a project-scoped server it asks for approval, so a cloned repository can't launch processes without consent. Anthropic, *Connect to MCP servers* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/mcp-quickstart](https://code.claude.com/docs/en/mcp-quickstart)

11. MCP servers must not accept tokens not explicitly issued for them; passing tokens through without validating claims such as audience lets a stolen token use the server as a proxy and bypasses audience-based controls. Model Context Protocol, *Security Best Practices* (accessed 2026-09-09; not re-checked for this edition), [https://modelcontextprotocol.io/specification/draft/basic/security_best_practices](https://modelcontextprotocol.io/specification/draft/basic/security_best_practices)

12. Anthropic reviews connectors against listing criteria but does not security-audit or manage MCP servers; administrators can deploy a fixed set through managed configuration, disable MCP, or filter with allowlists and denylists. Anthropic, *Control MCP server access for your organization* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/managed-mcp](https://code.claude.com/docs/en/managed-mcp)

13. Changes in MCP 2026-07-28 since 2025-11-25: all results carry a required `resultType` (`"complete"` or `"input_required"` for multi round-trip requests); every request carries its protocol version and client capabilities in `_meta`; servers must implement `server/discover`; version mismatches return `UnsupportedProtocolVersionError`; Roots, Sampling and Logging are deprecated; the HTTP+SSE transport is reclassified as Deprecated, with migration to Streamable HTTP; a feature lifecycle policy sets a minimum twelve-month deprecation window. Model Context Protocol, *Changelog, 2026-07-28* (accessed 2026-11-27), [https://modelcontextprotocol.io/specification/2026-07-28/changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)

14. The 2026-07-28 release makes the protocol stateless, so any request can land on any server instance behind a round-robin load balancer without shared storage; it adds header-based routing, cacheable list results, multi round-trip requests and authorization hardening; replacements for deprecated features are tool parameters or resource URIs for Roots, direct LLM provider integration for Sampling, and stderr or OpenTelemetry for Logging. Model Context Protocol Blog, *The 2026-07-28 Specification* and *Release candidate* (accessed 2026-11-27), [https://blog.modelcontextprotocol.io/posts/2026-07-28/](https://blog.modelcontextprotocol.io/posts/2026-07-28/); [https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)

15. MCP 2026-07-28 moves MCP to a stateless request/response core, and support is being rolled out across Claude products. Anthropic, *Bringing MCP 2026-07-28 to Claude* (Claude blog, accessed 2026-11-27), [https://claude.com/blog/bringing-mcp-2026-07-28-to-claude](https://claude.com/blog/bringing-mcp-2026-07-28-to-claude)
