# Chapter 15: The Constraints Nobody Mentions Decide the Design

**Summary**: *Every system you're asked to change already has properties nobody put in the request. A cache dropped into a multi-tenant authorisation path breaks one of them quietly, months later, once. This chapter covers the two sets of requirements between a business goal and a design you can defend: behaviours someone can mark pass or fail, and deployment constraints that only come out because someone asked. Find out about the second set late, and the fix is a migration, not a setting.*

## The Request Describes the Change; the System Holds the Constraints

A ticket arrives: **put a cache in front of the authorisation module**, because the checks are slow and the page waits on them.

Read it twice and nothing seems missing. There's a component, a symptom, a mechanism and an owner.

Now look at the **system** instead of the ticket. **Several customers share** the deployment. Every access decision is written to a **log someone audits**. The existing invalidation **doesn't behave the same on every path**, which the team knows and has never written down.

None of that was put there for this change. It was true last quarter and will be true after the cache ships. This book calls it a **standing constraint**: a property of the environment that **predates the request**, appears in **nobody's description** of the work, and **decides the design** anyway.

A missing feature announces itself the first time someone uses it. A standing constraint broken by a cache announces itself when **one tenant is shown another tenant's access decision**, months later, once. **Silent, rare and expensive** is exactly the kind of defect iteration never catches, because iteration needs a signal, and this one gives none until it gives a very big one.

So your first move on a change like this is **neither code nor a design**. It's a set of **targeted questions**:

- What has to **stay true**? (Can a cached decision for tenant A ever be served to tenant B?)
- What **invalidates an entry**? (A role change? A revoked key? How fast must it take effect?)
- What must the **audit log** record? (Is a cache hit still an access decision that must be logged?)
- What happens when the **cache is wrong**? (Fail open or fail closed?)

Anthropic makes the same point about Claude: jumping straight to code produces code that **solves the wrong problem**, so exploration and planning are kept separate from execution.<sup>[1]</sup>

## A Goal You Can't Fail Isn't a Requirement Yet

A **business problem** is a direction: "Get invoices approved sooner." "Stop the compliance team reading every document." Both are worth having, and you can **neither build nor test** a system against either one.

A **functional requirement** says what the system must do, precisely enough that someone can **mark it pass or fail**. Deriving requirements turns one direction into several **checkable behaviours**:

- Flag each document as needing review or not.
- Quote the clause that triggered each flag.
- Keep a flagged document unapproved until a named reviewer clears it.

Each of those can be run and scored. The direction can't.

The test is mechanical: read the statement and ask **what observation would show it had been broken**. If nothing would, it's a **preference**, not a requirement.

Anthropic uses the same discipline for success criteria. "Good performance" becomes "accurate sentiment classification". "Safe outputs" becomes "fewer than **0.1% of outputs across 10,000 trials** flagged for toxicity by the content filter".<sup>[2]</sup> The second version is longer, but what it buys is a line the system can be held to.

> **Example.** "The summariser should be fast and accurate" can't fail. "95% of summaries return within 4 seconds at the 95th percentile, and on our 200-ticket test set, a reviewer marks at least 90% as containing every action item" can.

The **requirement nobody can fail** is the mistake here, and it's tempting because it's agreeable. Everyone signs off on "fast, accurate and secure", and nothing has been agreed. Two people read one sentence, picture different systems, and discover the difference at the review where one says "this isn't what I asked for."

## The Second Set of Requirements Is on Other People's Desks

**Functional requirements arrive more or less on their own.** Whoever wants the feature knows what it should do and will tell you unprompted.

**Infrastructure requirements limit the deployment, not the behaviour.** They say nothing about what the system produces, and everything about the **conditions under which producing it is acceptable**. Almost none of them appear in the business problem. You get them by **asking**, not reading, because each one is obvious to the person who holds it, so they never say it. Four questions cover most of the ground:

| The question | Who actually knows | What a late answer costs |
|---|---|---|
| How fast, measured where the user is? | Whoever owns the user experience, and whoever runs the region | A model, platform or topology chosen against the wrong number |
| What volume, and what does the peak look like? | Operations, plus whoever has the historical figures | Capacity and rate limits sized for the average |
| Processed where, stored where, under which regulation? | Legal, compliance, the data protection team | A rebuild, because some settings are fixed at creation |
| Acting with whose credentials, and audited how? | Security, and the owner of the system being acted on | An audit trail that can't answer the question it exists for |

The third column is why this work belongs **at the start**. Four questions cost a conversation. Their answers, arriving late, cost a migration. (Chapter 4 covers trading off against a latency figure once you have it; this chapter is about getting it at all.)

One trap looks like diligence. A **latency budget tells you nothing about which tenant may see which record**. Performance targets and correctness rules are different kinds of statement, and inferring correctness from a performance target is guessing with a spreadsheet open.

## A Residency Rule Removes Options; It Never Gets Weighed Against Them

**Latency, scale, residency and identity** decide deployments more often than anything else, and they don't all behave the same way once design starts.

- A **latency target is a threshold.** Designs are measured against it; some pass; the ones that pass are then compared on other grounds.
- A **residency rule isn't a threshold.** It **removes candidates** before any comparison begins, and whatever survives is compared as if the rule had never come up.

On the Claude API, **where inference runs** and **where data is stored** are **two independent settings**:<sup>[3]</sup>

- **Inference geo** is set per request with `inference_geo` (`"us"` or `"global"`), or as a workspace default. It's supported on **Claude 4.6 and later**; older models return a `400` if you send it. US-only inference costs **1.1×** standard prices, across input, output, cache writes and cache reads. The Batch API accepts it per request.
- **Workspace geo** controls where data is **stored at rest** and where endpoint processing (such as image transcoding and code execution) happens. It's **set when the workspace is created and can't be changed**. Currently `"us"` is the only option.

> **Example.** A healthcare customer requires all processing in the US. Setting `default_inference_geo: "us"` on the workspace covers inference. But if the workspace was created without a US workspace geo, its storage location can't be changed; you need a **new workspace**. Learning this in week one is a setting; learning it in month six is a migration.

**Retention** works the same way, and bites harder:<sup>[4]</sup>

- **Zero data retention (ZDR)** is arranged **per organisation** through Anthropic's account team. Enabling it for one organisation doesn't extend it to another under the same account.
- **HIPAA readiness** needs a **signed business associate agreement**, and once enabled it's **permanent**; no administrator can turn it off.
- A request that uses a **feature the arrangement doesn't cover fails with a `400`**, rather than quietly degrading. Some features, such as Managed Agents (chapter 13), aren't covered at all.
- On **Amazon Bedrock and Google Cloud**, the cloud provider is the data processor, so **their** retention documentation applies.

**Treating residency as one factor to balance** is the standard wrong answer, and it always looks like engineering judgement: a faster region, a cheaper platform, a simpler integration. None of those is an argument, because the constraint decides which options are **allowed at all**, and the ones being weighed stopped being options before anyone weighed them.

## Write It Down, Because the Reviewer Wasn't in the Room

Requirements get written down for one reason that survives a busy team: **whoever reviews a deployment choice wasn't in the conversation** where its constraints came out, and the review is where the choice has to be defended.

A short record is enough:

- the **behaviours** the system must show;
- the **infrastructure constraints**;
- **where each constraint came from** (which regulation, which owner).

With it, your platform choice reads as a **consequence of the requirements**. Without it, it reads as **habit**, and habit is exactly what a reviewer pushes on.

The record must **stay current**, because configuration doesn't update itself retroactively. In Managed Agents, tightening a permission policy governs **sessions created after the change**; sessions already running **keep their original configuration**.<sup>[5]</sup> Self-hosting has the same asymmetry: once session content reaches your worker, keeping, redacting or deleting it is **your** policy, and Anthropic has no visibility into what happens next.<sup>[6]</sup>

None of this applies to every ticket, and pretending it does is how teams abandon the discipline altogether. Anthropic's calibration is worth borrowing: planning earns its overhead when the approach is **uncertain**, the change touches **several files**, or the code is **unfamiliar**. A change whose diff fits in one sentence should skip it.<sup>[1]</sup> **Multiple tenants, an audit requirement and behaviour nobody describes the same way twice** clear that bar three times over.

## How the Exam Builds Wrong Answers

Wrong answers here describe good engineering: prototyping, staged rollout, monitoring and reuse. What makes one wrong is **what it assumes is already known**.

- **The unstated property assumed absent.** The option treats the system as if the only true facts are the ones the request mentioned. Iterating, shipping behind a flag, and building a small version first all belong to a system whose invariants are known. Each becomes correct once the change is **reversible** and the damage is **loud and contained**.
- **Discovery moved to after the build.** Monitoring, incident review and canary releases do find constraints, **by breaking one**. That's a fair trade when failure is cheap, quick and visible; a bad trade when it's **silent, rare and consequential**, which is the profile of every tenancy and audit scenario.
- **A constraint guessed instead of asked for.** Reading correctness rules off a performance target, a neighbouring module, or what the last team wrote. Copying an approach **imports its assumptions** too, and an authorisation path is where imported assumptions do the most damage.
- **A hard constraint weighed as a preference.** A regulation, jurisdiction or retention rule offered as one input to a trade-off against cost, latency or effort. Those constraints **produce the list of candidates**; they never compete inside it.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| Properties the request didn't mention: several tenants, an audit trail, behaviour that varies by path | Ask about invariants, invalidation, the audit obligation and failure modes first | Iterating, prototyping or a feature flag, which learn the same facts from an incident |
| A goal stated as faster, better, accurate or seamless | Turn it into behaviours someone can mark pass or fail | Accepting the goal as the requirement and moving on to mechanism |
| A regulation, jurisdiction or retention rule appears anywhere | Use it as the filter that produces the candidate list | Balancing it against latency or cost |
| Behaviour is fully specified and a platform decision is due | Get latency, volume, residency and identity, then decide | Choosing by familiarity, existing tools, or what shipped last time |
| Someone proposes copying an approach from a neighbouring system | Check which of that system's assumptions hold here | Reuse justified by similar shape, not shared invariants |
| A design makes sense inside the team but not outside it | Write the record, naming the source of every constraint | Documenting the decision without the constraints that forced it |

One habit works beyond the table. For each option, ask **what it needs you to know already**, then check whether the scenario said that was **settled** or said **nobody had established it**.

## What to Remember

- A **standing constraint** is a property of the environment that **predates the request**, isn't mentioned, and **decides the design**. The usual three: **multiple tenants**, an **audit obligation**, and **inconsistent existing behaviour**.
- **Ask before building** wherever a mistake would be **silent, rare and expensive**. Iteration needs a signal; a tenancy or authorisation bug gives none until it gives a big one.
- A **functional requirement** can be **marked pass or fail**. Test it by asking what observation would prove it broken; if none, it's a **preference**.
- **Infrastructure requirements** limit the deployment, not the behaviour, and come from **asking**: **latency at the user**, **peak volume**, **processing and storage location under a named regulation**, and **whose credentials act, with what audit record**.
- **Residency, retention and identity rules decide which options are allowed.** Speed, cost and familiarity only rank what survives.
- **Inference location and storage location are separate.** `inference_geo` is per request or workspace default (1.1× for US-only, Claude 4.6+). **Workspace geo is fixed at creation**, so a late residency answer costs a **rebuild**.
- **ZDR is per organisation; HIPAA readiness is permanent once on**; uncovered features return a `400`.
- **A performance target implies no correctness rule.** A latency budget says nothing about which tenant may see which record.
- **Write requirements down** for the reviewer who wasn't there: behaviours, constraints, and the **source** of each constraint. That makes a platform choice a **consequence**, not a habit.

## Endnotes for this chapter

1. Jumping straight to code can produce code that solves the wrong problem, so plan mode separates exploration from execution; planning helps when the approach is uncertain, the change spans multiple files, or the code is unfamiliar, and a change describable in one sentence should skip it. Anthropic, *Best practices for Claude Code* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)

2. Good success criteria are specific and measurable: "good performance" becomes "accurate sentiment classification", and "safe outputs" becomes less than 0.1% of outputs out of 10,000 trials flagged for toxicity. Anthropic, *Define success criteria and build evaluations* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/test-and-evaluate/develop-tests](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests)

3. Two independent settings govern data residency: inference geo, set per request with `inference_geo` or as a workspace default (`default_inference_geo`), and workspace geo, which controls where data is stored at rest and where endpoint processing such as image transcoding and code execution happens. Workspace geo is set at creation and can't be changed; `"us"` is currently the only workspace geo. For Claude 4.6 and later, `inference_geo: "us"` costs 1.1x standard pricing across all token categories, `"global"` uses standard pricing, and older models return a 400 if the parameter is sent. The Batch API supports `inference_geo` per request. Anthropic, *Data residency* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/manage-claude/data-residency](https://platform.claude.com/docs/en/manage-claude/data-residency)

4. ZDR is enabled per organisation by an account team and doesn't extend to other organisations under the same account; HIPAA readiness requires a signed business associate agreement, is permanent once enabled, and requests using uncovered features return a 400; Claude Managed Agents is covered by neither; on Amazon Bedrock and Google Cloud the cloud provider is the data processor. Anthropic, *API and data retention* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/manage-claude/api-and-data-retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention)

5. Running sessions keep the toolset configuration they were created with; permission policy updates apply to sessions created afterwards. Anthropic, *Permission policies* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/permission-policies](https://platform.claude.com/docs/en/managed-agents/permission-policies)

6. For self-hosted sandboxes, session content passes through the operator's worker, and retaining, redacting or deleting it is the operator's responsibility; Anthropic has no visibility into what the worker does with it. Anthropic, *Security model* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes-security](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes-security)
