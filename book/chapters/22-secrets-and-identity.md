# Chapter 22: One Credential: What It Reaches, and for How Long

**Summary**: *Think of a secret as a set of copies, not a single value. Each copy has its own reach and its own remaining lifetime. Three measurements decide what a leak costs, and every practice in this skill changes exactly one of them: how many copies exist, what one copy can do, and how long it keeps working after someone else gets it. A fourth question decides whether anyone ever finds out, and it's the one with no symptom when the answer is "nobody".*

## The Value Is Shown Once, Then Ends Up Wherever It Was Needed

Create an API key in the Claude Console. The **full secret is shown once**, at creation. If you lose it, you **can't view it again**; you create a new one.<sup>[1]</sup> That single display is a decision about **copies**: the platform holds nothing you can read back, so every copy from then on exists because **someone made it**.

Follow one key through a week of setup:

1. It goes into a **shell profile** so a local run works.
2. A **deployment config** gets a copy.
3. Someone **pastes it into a chat** to unblock a colleague.
4. A **CI runner caches** it.
5. A **backup** captures the runner.
6. A teammate puts it in their own **`.env`**.

**Six copies**, none wrong when made, and **no list** of where they are. So the only question that decides what a leak costs: what does any one of those six **reach**, and how long does it **keep working**?

## Three Measurements Decide the Damage

Call that list the **copy set**: every place the credential's value has ended up, with the access each place has. Every decision in this chapter acts on one of three measurements:

- **Count**: how many copies exist, and who can read each.
- **Reach**: what one copy **authorises**.
- **Life**: how long it **keeps working** once someone else has it.

| Practice | Measurement it changes | What it leaves untouched |
|---|---|---|
| Keeping the value out of files that reference it | **Count** | Reach and life of the remaining copies |
| A managed secret store read at runtime | **Count**, plus a record of reads | Reach |
| Separate, narrowly scoped credentials per environment and service | **Reach** (down) | Count, which goes up |
| An expiration set at creation, plus scheduled rotation | **Life** | Count and reach |
| IP allowlisting | **Reach** of a leaked copy (where it works) | Count and life |
| Federation (short-lived tokens instead of keys) | **Life** (minutes) and **count** (no durable secret) | Reach |
| Access logs and usage alerts | **None** of the three | All three |

Read the **right-hand column**. Every practice leaves something untouched, which is why an option naming one practice and stopping answers a **narrower question** than a credential review asks.

## A Value Written Into a Tracked File Survives Its Own Deletion

**Separation** means a config file holds a **reference**, never a **value**. This isn't tidiness. A tracked, cloned, reviewed or backed-up file carries its contents to all those places. **Overwriting** the value later changes only the **current version**; git history keeps the old one, in **every clone made before the fix**. So the anti-pattern is **overwrite mistaken for removal**, and any credential ever written inline must be treated as **exposed and replaced**.

> **Example.** A developer commits `config.yaml` with `api_key: sk-ant-api03-...`, notices an hour later, and pushes a commit replacing it with `api_key: ${ANTHROPIC_API_KEY}`. The key is still in the first commit, in every fork and every CI checkout from that hour. The right response is to **revoke the key** and create a new one, then make the reference change.

The value needs a home. Two choices:

- An **environment variable injected at run time** suits a secret that's **local and short-lived**: one process, one run, nothing on disk.
- A **managed secret store** suits a secret **several services need**. It **collapses scattered copies into one** and **records who read it**.<sup>[1]</sup>

Stronger still is **taking the value out of the workload entirely**:

- A **proxy outside the agent's boundary** injects credentials into outgoing requests, so the agent never sees them.<sup>[4]</sup>
- **Managed Agents vaults** store an opaque **placeholder** in the sandbox and swap in the real secret **at egress**.<sup>[7]</sup>
- In Claude Code, **`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB`** strips provider credentials from the environments of **subprocesses** (Bash, hooks, MCP stdio servers), so a prompt injection can't read them through shell expansion.<sup>[8]</sup>

## Better Still: No Long-Lived Key at All

Anthropic now treats **workspace API keys as legacy**. The recommended options are:<sup>[2]</sup>

| Method | Credential | Best for |
|---|---|---|
| **Personal API key** | A static key linked to **you** | Your own development |
| **Service account key** | A static key linked to a **service account** | Anything shared across people or systems |
| **Workload Identity Federation (WIF)** | A **short-lived token** exchanged from your identity provider's token | Production workloads on AWS, Google Cloud, Azure, Kubernetes, and CI such as GitHub Actions |
| **App Attest** | Device attestation | iOS and macOS apps distributed to end users |

**Service accounts** give each workload its **own identity, roles and audit trail**, instead of everyone sharing one key.<sup>[2]</sup>

**Federation** removes the durable secret rather than shortening its life. A workload trades its platform-issued identity token for a **Claude API access token that expires in minutes**, and the SDK refreshes it automatically. **Nothing static is minted, distributed or rotated.** The chain is only as strong as the **identity provider's configuration**, so pair it with the IdP's own controls (workload identity binding, conditional access, audit logs).<sup>[2]</sup>

> **Example.** A GitHub Actions workflow calls Claude to review pull requests. With WIF, the job exchanges GitHub's OIDC token for a short-lived Claude token, so there's **no `ANTHROPIC_API_KEY` secret** in the repository at all. One trap: **`ANTHROPIC_API_KEY` takes precedence over federation**, so a leftover key in a container env, CI secret or shell profile silently shadows WIF. Remove it everywhere, and confirm with `ant auth status`.<sup>[2]</sup>

## Some Properties Are Fixed When the Key Is Created

**Reach** is decided twice: by **how many things** one credential serves, and by what it's **permitted** to do.

A credential shared across development, staging and production is a **path between environments you separated on purpose**. It turns **one compromise into three**, and it can't be replaced anywhere without replacing it everywhere. **Separate credentials per environment and per service** raise the count deliberately and shrink what any one is worth to an attacker.

Several key properties are **fixed at creation**:<sup>[1]</sup><sup>[2]</sup><sup>[3]</sup>

- A key's **workspace**, its **expiration**, and any **scopes** the product lets you choose can only be changed by **issuing a new key**.
- **Console Admin API keys** have **no selectable scopes**: each one has **full access** to every endpoint that accepts admin keys. A key from one organisation can't manage another.
- Only an **organisation admin** can create an admin key.

So **issuing a key is the whole authorisation decision**, and approval belongs **at issue time**.

To limit **where** a leaked key works, turn on **IP allowlisting**.<sup>[2]</sup>

## Rotation Is the Only Repair; Expiry Is the Version That Runs by Itself

A **leaked value can't be made secret again**. The only repair is to **issue a replacement and revoke the old one**. Anthropic's guidance: keep keys in a **secrets manager**, **rotate periodically**, and **revoke any key you suspect has leaked**.<sup>[2]</sup><sup>[5]</sup>

**Scheduled rotation** and **rotation on suspicion** do different jobs:

- The **schedule** limits how long an **undetected** leak lasts.
- **Immediate rotation** is incident response. It gets cheaper as the copy set shrinks, because a consumer that reads the secret **by name** from a store needs no change when the value behind the name is replaced.

Rotation that **puts the new value back where the old one leaked from** wastes both: it clears the error and **rebuilds the exposure**.

**Expiry** is rotation's automatic cousin. You choose it at creation (a preset, a custom duration, or never), and **it can't be changed afterwards**. Once it passes, requests return **`401`** and the key **can't be reactivated**; expired keys can only be deleted. The Admin API reports each key's `expires_at`, so you can audit and rotate ahead of time. Expiry **limits a leaked key's usable life**; it **doesn't replace** good storage or rotation.<sup>[2]</sup>

## A Verified Identity Is Still Just a Claim About Who

Sessions in a **self-hosted environment** carry a **signed session token** that your own services can check. A valid token proves Anthropic issued it for a **specific session in a specific environment**, and how that session was created. **It proves nothing more.** It doesn't prove **which process** presented it, because it sits in an **environment variable** any code in the session can read.<sup>[6]</sup>

So verification is mechanical:<sup>[6]</sup>

- Check the **signature** against Anthropic's published key set.
- **Reject unexpected algorithms.**
- Check the **audience** claim against **your own environment ID**, so tokens issued to another organisation's environment are rejected.

Verification is **offline**: a token that verifies **stays valid until it expires**, whatever has happened to the session since, and there's **no revocation feed**.

Then the part identity doesn't settle. The anti-pattern is **identity accepted as authorisation**: a verified caller gets **everything that caller holds elsewhere**. The token names the person who **created** the session; it isn't that person **signing in**. Credentials derived from it should cover only **what one session needs**, and expire **no later than the token**.<sup>[6]</sup>

> **Example.** An internal deploy service receives a request with a valid session token created by Priya, who is a production admin. The service should grant this session **read access to the build logs it needs**, valid for at most the token's remaining life, not Priya's full production admin rights.

## Nobody Outside Your Organisation Can Tell You a Key Leaked

Anthropic states the limit: it can **detect anomalous usage patterns**, but it **can't know that a particular key was compromised**.<sup>[5]</sup> That knowledge is produced **locally or not at all**. Three records answer three questions:

- A **managed store logs who read** a value, which narrows the copy set after the fact.
- The **Admin API lists every key** with its expiration, returning a **redacted hint, never the secret**, so you can audit the inventory and rotate before expiry.<sup>[1]</sup><sup>[2]</sup>
- Where work runs under a derived credential, logging the **session ID next to the human identity** traces one action to **one session**, not just to a busy person.<sup>[6]</sup>

The anti-pattern is the **credential nobody watches**, and it **fails quietly**. Every other control here shows itself when it refuses something. Missing monitoring **looks exactly like success**.

## How the Exam Builds Wrong Answers

Wrong answers here **describe real practices**. Each improves something, so you can't eliminate options as obvious nonsense.

- **One measurement improved, the others ignored.** A key moved out of source control into **plaintext build config**. A shared credential **rotated on schedule but still shared**. Each is an improvement and leaves the copy set as wide as before. Right when the question is **what to fix first**; wrong when it asks for the **strongest posture**.
- **One shared object sold as an advantage.** One credential everywhere really is simpler. That's also the **defect, stated out loud**. Ask what a single replacement costs and how many environments one compromise crosses.
- **A control that acts after the value has travelled.** Deleting the committed value, restricting networks, tightening review. None reduces the copies **already made** or shortens the life of the one someone took.
- **A verified identity spent as authorisation.** A caller with a valid token is handed everything they hold elsewhere. Verification answers **who**, never **what may be done**.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| Review credential handling across several environments | Change all three measurements: separate, scope narrowly, store centrally, rotate on schedule and on suspicion | One practice named as enough, or one shared key defended as simpler |
| A credential is in a tracked config file | Treat it as exposed: revoke, replace, and leave a reference where the value was | Overwriting the value in a new commit |
| One key serves dev, staging and production | Separate credentials per environment and service | Rotating the shared key more often |
| Several services need a key; where to keep it? | A managed store, or config outside source control, loaded at run time | Anywhere convenient that keeps a permanent copy |
| A CI pipeline or cloud workload needs Claude API access | Workload Identity Federation, with no static key | A long-lived key stored as a CI secret |
| WIF is configured but requests still use the old key | Remove `ANTHROPIC_API_KEY` everywhere; it takes precedence | Reconfiguring the federation rule |
| A request arrives with a signed session token | Verify signature, algorithm and audience; grant only what that session needs | Granting the named user's full access |
| Has a credential been misused? | Check records: store read logs, session IDs, key inventory | Assuming the platform would have said so |

One habit works beyond the table: for any option, **name which measurement it changes** and which it leaves alone. The strongest answer is nearly always the one that **changes all three**.

## What to Remember

- A **credential is a copy set**. Three measurements: **count** (how many copies), **reach** (what one copy can do), **life** (how long it works).
- A value in a **tracked file is in history**. Overwriting edits only the current version. **Revoke and replace** any inline credential.
- **Config holds references, not values.** Env vars for one process and one run; a **managed store** for shared secrets (fewer copies, read logs). Better: a **proxy**, **vault placeholders**, or **subprocess credential scrubbing**, so the workload never holds the value.
- **Workspace API keys are legacy.** Use **personal keys**, **service account keys**, or **Workload Identity Federation** (short-lived tokens, no static secret). A leftover **`ANTHROPIC_API_KEY` overrides WIF**.
- **Separate credentials per environment and service.** A shared key links environments you separated on purpose.
- A key's **workspace, expiration and scopes are fixed at creation**. **Admin API keys have full access**, no scopes. Approval belongs at **issue time**. **IP allowlisting** limits where a leaked key works.
- **Rotation is the only repair.** Rotate on a **schedule** and **on suspicion**; never put the new value back where the old one leaked. **Expiry** limits a leak's life; expired keys return **`401`** and can't be reactivated.
- A **verified identity is authentication, not authorisation.** Check **signature, algorithm and audience**; scope derived credentials to **one session**, expiring no later than the token.
- **Anthropic can see anomalous usage but can't know a key leaked.** Your **read logs, key inventory and session IDs** turn assumptions into evidence.

## Endnotes for this chapter

1. The Claude Console shows a new API key's full value only once, at creation, and a lost key can't be viewed again; keys are created with a name, expiration and linked account; the recommended storage is a secrets manager; the Admin API manages keys programmatically, requires a separate Admin API key, and returns only a redacted hint, never the secret. Anthropic, *Get your Claude API key* (Claude Platform Docs, accessed 2026-09-10; not re-checked for this edition), [https://platform.claude.com/docs/en/get-api-key](https://platform.claude.com/docs/en/get-api-key)

2. The Claude API supports API keys, Workload Identity Federation and App Attest, and API keys and WIF grant the same access; workspace API keys are legacy, and identity-backed keys (a personal key linked to yourself, or a service account key for shared use) or WIF are preferred; expiration is chosen at creation and can't be changed afterwards, expired keys can only be deleted, and the Admin API reports each key's `expires_at`; IP allowlisting limits where a leaked key works; WIF lets workloads exchange short-lived OIDC tokens from GitHub Actions, AWS, Google Cloud, Microsoft Entra ID or Kubernetes for short-lived Claude API access tokens that expire in minutes, should be paired with the identity provider's own controls, and is only as strong as the upstream provider; `ANTHROPIC_API_KEY` takes precedence over federation, so a leftover key silently shadows it, and `ant auth status` reports which source won; keys should be stored in a secrets manager, rotated periodically, and revoked if suspected of leaking. Anthropic, *Authentication* and *Workload Identity Federation* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/manage-claude/authentication](https://platform.claude.com/docs/en/manage-claude/authentication), [https://platform.claude.com/docs/en/manage-claude/workload-identity-federation](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation)

3. Only organisation members with the admin role can create Admin API keys; Console Admin API keys have no selectable scopes and carry full access to all endpoints that accept them; a key from one organisation can't manage another; where scopes are selectable they're fixed at creation. Anthropic, *Create an Admin API key* (Claude Platform Docs, accessed 2026-09-10; not re-checked for this edition), [https://platform.claude.com/docs/en/manage-claude/admin-api-keys](https://platform.claude.com/docs/en/manage-claude/admin-api-keys)

4. Sensitive resources such as credentials can be placed outside the agent's security boundary; a proxy outside the boundary can inject credentials into outgoing requests so the agent never sees them. Anthropic, *Securely deploying AI agents* (Claude Code Docs, accessed 2026-09-10; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/secure-deployment](https://code.claude.com/docs/en/agent-sdk/secure-deployment)

5. The self-hosted environment service key should be kept in a secrets manager and rotated immediately if exposure is suspected; separate workspaces and environments per trust boundary limit each key's reach; Anthropic can detect anomalous usage but can't know a customer's key was compromised, so a suspected leak must be revoked and replaced by the customer; revocation takes effect on the next request. Anthropic, *Security model* (Claude Platform Docs, accessed 2026-09-10; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes-security](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes-security)

6. A valid session token proves Anthropic issued it for a specific session in a specific environment, not which process presented it, since it sits in an environment variable any code in the session can read; services must verify the signature against Anthropic's published key set, reject unexpected algorithms and verify the audience claim; verification is offline, tokens stay valid until expiry, and there's no revocation feed; derived credentials should be limited to one session's needs, bounded by the token's expiry, and audited with session and token IDs alongside the user identity. Anthropic, *Verify session identity in self-hosted environments* (Claude Code Docs, accessed 2026-09-10; not re-checked for this edition), [https://code.claude.com/docs/en/self-hosted-environments-identity](https://code.claude.com/docs/en/self-hosted-environments-identity)

7. Vault credentials of the environment-variable type are stored in the sandbox as an opaque placeholder and replaced with the real secret at egress, so the agent never sees the value. Anthropic, *Authenticate with vaults* (Claude Platform Docs, accessed 2026-09-10; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/vaults](https://platform.claude.com/docs/en/managed-agents/vaults)

8. `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` strips Anthropic and cloud provider credentials from subprocess environments (Bash, hooks, MCP stdio servers) while the parent process keeps them for API calls. Anthropic, *Environment variables* (Claude Code Docs, accessed 2026-09-10; not re-checked for this edition), [https://code.claude.com/docs/en/env-vars](https://code.claude.com/docs/en/env-vars)
