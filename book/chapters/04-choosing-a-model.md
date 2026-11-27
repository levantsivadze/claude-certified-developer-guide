# Chapter 4: Choosing a Model Starts With Three Numbers

**Summary**: *Most feature requests say what to build but not how good, how fast or how cheap it must be. Without those three numbers, any model can be defended, so you can't make a real choice. This chapter covers how to get those numbers, how the current models compare, and why a model you pinned can still change under you.*

## The Ticket Is Missing Three Numbers

A typical ticket looks complete. It lists what the feature does, the acceptance criteria, the edge cases, the owner and the deadline. At the bottom, one line says: "Use whichever Claude model gives the best results."

That line hides three questions. Best by what standard? Within what wait? At what price?

This book calls them **the three targets**:

1. **Quality bar**: how good the output must be.
2. **Latency ceiling**: how long a user or system can wait.
3. **Cost per unit**: how much the business can spend per task.

You can't make a trade-off until you know what you're trading against. While all three are blank, every model is a reasonable answer. If you can argue for four different answers, the question is incomplete, not hard. People usually describe what a feature should do without being asked. You have to ask for the three targets.

> **Example: a ticket summariser**
>
> The ticket says: *"Summarise each incoming support ticket for the agent who picks it up. Use the best model."*
>
> After asking the support lead, you get the three targets:
>
> - **Quality:** in a sample of 200 tickets, agents rate at least 95% of summaries as "correct and useful".
> - **Latency:** the summary appears within 3 seconds of the ticket opening.
> - **Cost:** under $0.002 per ticket, because there are 50,000 tickets a day (a budget of $100 a day).
>
> Now the choice is real. You run the 200-ticket sample on Haiku 5.5 and Sonnet 5.5. If Haiku 5.5 hits 95% within 3 seconds and under budget, you're done. If it reaches only 89%, you try Sonnet 5.5 and check whether it fits the budget. You never had to argue about which model is "best".

Three shortcuts look like progress but skip this step:

- **Picking from a public leaderboard.** It measures someone else's tasks and sets no bar for yours.
- **Choosing the middle model "to be safe."** It guesses all three numbers and writes none of them down.
- **Choosing the top model by default.** It spends everything on quality and leaves speed and cost unplanned. This is the most common and most expensive mistake.

## The Current Lineup

Once you have the targets, the choice is short. Here are the current models, from cheapest and least capable to most expensive and most capable:<sup>[1]</sup>

| Model | Price per million tokens (input / output) | Speed | Best for |
|---|---|---|---|
| Claude Haiku 5.5 | from $0.10 / $0.50<sup>[2]</sup> | Fastest | High-volume work: classification, extraction, routing |
| Claude Sonnet 5.5 | $2 / $10 | Fast | A balance of speed and intelligence |
| Claude Opus 5.5 | $4 / $20 | Moderate | Long-running agentic coding and knowledge work. **The recommended default** |
| Claude Fable 5.2 | $8 / $40 | Moderate | Hard reasoning and long-horizon agents, or when Opus 5.5 at higher effort still falls short |

All four have a 1M token context window and up to 128K output tokens.<sup>[1]</sup> All four take text and images and support tool use.

A few things to notice:

- **Anthropic now recommends starting with Opus 5.5**, not the middle or the top. Move up to Fable 5.2 only when your evals on Opus 5.5 fall short. Move down to Sonnet 5.5 or Haiku 5.5 for speed and volume.<sup>[1]</sup>
- **Haiku 5.5 is priced by prompt length.** Above 100,000 input tokens, every token costs 5 times more. For long-running Haiku agents, compact the conversation before it passes that line.<sup>[2]</sup><sup>[3]</sup>
- **Older models have smaller limits.** Opus 4.5, Sonnet 4.5 and earlier have a 200K context window and 32K to 64K maximum output.<sup>[1]</sup> If you're maintaining an older integration, one long answer can rule a model out before quality even comes up.

**Names and prices change**, so check the lineup when you build. The pattern lasts longer than the list: higher tiers cost more per token and are more capable.

## Compare Cost per Finished Task, Not per Token

Per token, the bigger models look expensive. But you're paying for finished work, not tokens. A stronger model often finishes in fewer turns, with less searching and less backtracking, and that can cancel out its higher price.<sup>[3]</sup>

Some examples from Anthropic's own measurements on a coding benchmark:<sup>[3]</sup>

- Opus 5.5 at its default effort solved about as many tasks as Fable 5.1, at **about a fifth of the cost** per solved task.
- Opus 5.5 at low effort matched Sonnet 5.5 at its default for **only about 7% more** per solved task.

The result can go either way. On a research benchmark, Fable 5.2 cost almost 3 times as much as Opus 5.5 and scored slightly lower.<sup>[3]</sup> The only way to know for your workload is to measure it.

### Price the hardest tasks, not the typical one

On a typical request, every model looks about the same, so the cheapest one looks best. But your bill depends on the requests the model fails. A failed task still bills its tokens, then the retry, then whatever damage the wrong answer causes downstream.

In one Anthropic test of 20 problems, **just 2 problems made up 43% of the total spend**.<sup>[3]</sup> So look at the hardest 10% of your workload, not the average.

> **Example: the cheap model that costs more**
>
> *The numbers here are made up to show the arithmetic.* You process 100 invoices. A failed extraction goes to a person, which costs $2 of their time.
>
> | | Cheap model | Stronger model |
> |---|---|---|
> | Price per attempt | $0.02 | $0.05 |
> | Invoices handled correctly | 60 | 95 |
> | Token cost for 100 invoices | $2 | $5 |
> | Failures sent to a person | 40 × $2 = $80 | 5 × $2 = $10 |
> | **Real total** | **$82** | **$15** |
>
> Per token, the cheap model is 2.5 times cheaper. Per finished invoice, it's more than 5 times more expensive. Exam questions about cost often hide this kind of downstream cost in the scenario.

The cheapest model is a good fit when output is easy to check. On a graduate-level science test, Haiku 5.5 scored 85% against 91% for Opus 5.5, at about **1/20 of the cost per correct answer**.<sup>[3]</sup> That's a great deal when you can catch the wrong answers. It's a bad one when a wrong answer travels on unchecked.

### Let an eval decide

**Change models because a measurement says so, not because of an argument.** Move up when an eval shows the current model missing the bar on your hardest cases. Move down when an eval shows a cheaper model still meeting it. Chapter 19 covers how to build the eval.

Upgrading to a newer model is often the cheapest change of all. Going from Opus 5 to Opus 5.5 kept the score about the same and cut cost per solved task by about 78% on one coding benchmark.<sup>[3]</sup>

## Thinking Depth and Speed Are Separate Settings

Chapter 1 showed that **model, thinking and effort** are separate settings. For model choice, the key point is that **models don't all support the same settings**.

**Effort** controls how much the model thinks. The levels are `low`, `medium`, `high`, `xhigh` and `max`.<sup>[4]</sup>

- The default is `high` on most models, but **`medium` on Opus 5.5 and Haiku 5.5**.<sup>[4]</sup>
- A few older models lack the top levels. Opus 4.6, Opus 4.5 and Sonnet 4.6 don't have `xhigh`, and Opus 4.5 doesn't have `max`.<sup>[4]</sup>
- Lower effort is also a speed setting. On one research benchmark, low effort cut task time by 84%, but scored 13 points lower.<sup>[3]</sup>

**Thinking** works differently depending on the model:

- **Opus 5.5 and Fable 5.2 always think.** Sending `thinking: {"type": "disabled"}` or a manual `budget_tokens` value returns a `400` error. You control thinking depth with effort instead.<sup>[5]</sup><sup>[1]</sup>
- Sonnet 5.5 and Haiku 5.5 use adaptive thinking, where the model decides how much to think.<sup>[1]</sup>

### Fast mode: pay for speed, keep the same quality

**Fast mode** runs the same model on faster hardware. Quality and capabilities stay the same. You set `speed: "fast"` and get up to **2.5 times more output tokens per second**, at **twice the normal token price**. It's available on Opus 5.5 and Fable 5.2.<sup>[3]</sup><sup>[6]</sup>

So when speed is the problem and quality is fine, use something that leaves the output alone: lower effort, streaming or fast mode. **Switching to a smaller model is not that kind of fix**, because it changes what you get back.

> **Example: a slow coding assistant**
>
> Users of your in-editor assistant on Opus 5.5 say the answers are good but take 15 seconds to appear. Here are four possible fixes:
>
> | Fix | Effect on speed | Effect on answers |
> |---|---|---|
> | Stream the response | First words appear in about a second | None |
> | Turn on fast mode | Up to 2.5× faster output | None, but tokens cost twice as much |
> | Lower effort from `medium` to `low` | Less thinking, so faster | Slightly worse on hard problems. Check with an eval |
> | Switch to Haiku 5.5 | Much faster | Different model, so the quality users liked may be gone |
>
> Since users like the answers, start with streaming. Add fast mode if it's still too slow and the budget allows it. Switching models is the last option, not the first.

## A Pinned Model ID Doesn't Freeze Everything

Starting with the 4.6 generation, model IDs have no date in them, for example `claude-opus-5-5`. Each one points to **one fixed snapshot**. The weights behind an ID never change; an update ships under a new ID.<sup>[7]</sup> A common mistake is thinking a dateless ID always points to the newest model. It doesn't.

Older models are different. Their short names, such as `claude-opus-4-5`, are aliases that point to the latest dated snapshot, such as `claude-opus-4-5-20251101`.<sup>[7]</sup> So two IDs that look alike can promise opposite things.

| ID | What it means | Can the model behind it change? |
|---|---|---|
| `claude-opus-5-5` | Dateless ID (4.6 generation and later). It *is* the snapshot | No |
| `claude-opus-4-5-20251101` | Dated ID for an older model | No |
| `claude-opus-4-5` | Alias for an older model, pointing to its latest dated snapshot | Yes, if a newer snapshot ships |

Even with a pinned ID, the system around the model still changes. The routing, safety classifiers and sampling logic get updated, and that can cause small behaviour changes on an ID nobody touched.<sup>[7]</sup>

You also can't stay on one model forever. Models get deprecated, then retired, with **at least 60 days' notice**. After the retirement date, **requests fail**. Partner platforms such as Amazon Bedrock and Google Cloud set their own dates.<sup>[8]</sup> For example, Sonnet 4.5 retires on the Claude API on November 30, 2026, and Opus 4.5 on January 22, 2027.<sup>[9]</sup>

### Upgrading is more than changing a string

The request format survives an upgrade, but several other things may not.

**Some changes fail loudly with a `400` error.** Moving to Opus 5.5:<sup>[5]</sup>

- Disabling thinking or setting `budget_tokens` is rejected.
- Forcing a tool call with `tool_choice` set to `any` or a named tool is rejected. Use `auto` and say in the prompt when to use the tool, or use structured outputs.
- Editing earlier messages, the system prompt or tools before a thinking block can fail. Keep the conversation append-only.

Here's the same request before and after moving from Opus 5 to Opus 5.5:

```python
# Worked on Claude Opus 5, returns 400 errors on Claude Opus 5.5
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=4000,
    thinking={"type": "disabled"},                  # 400: thinking is always on
    tool_choice={"type": "tool", "name": "lookup"}, # 400: forced tool use
    tools=[lookup_tool],
    messages=messages,
)

# Works on Claude Opus 5.5
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,                    # leave room for thinking
    thinking={"type": "adaptive"},       # or leave the field out
    output_config={"effort": "medium"},  # set effort explicitly
    tool_choice={"type": "auto"},        # say in the prompt when to use the tool
    tools=[lookup_tool],
    messages=messages,
)
```

Since Opus 4.7, setting `temperature`, `top_p` or `top_k` to a non-default value and prefilling the assistant message also return `400` errors.<sup>[10]</sup>

**Some changes break nothing, but cost or behave differently:**

- Opus 5.5 defaults to `medium` effort where Opus 5 ran at `high`. Set effort explicitly and re-run your evals.<sup>[5]</sup>
- Opus 5.5 thinks more per turn at the same effort level. Leave room for thinking in `max_tokens`.<sup>[5]</sup>
- On Opus 5.5, text between tool calls arrives in `thinking` blocks. An app that streams that text as progress updates can go silent with no error.<sup>[5]</sup>
- Claude 4.7 and later models use about 30% more tokens for the same text.<sup>[3]</sup>

**The quietest problem is old prompts.** Prompts often collect instructions like "verify twice" or "be maximally thorough" that an older model needed. Newer models follow them literally, which adds tool calls and cost. In Anthropic's support-desk test, moving from Opus 4.8 to Opus 5 with the same prompt raised the cost. After a prompt audit, the same work was cheaper and more accurate.<sup>[3]</sup> Nothing breaks; the bill just goes up. That's why a model choice needs regular upkeep. Chapter 18 covers pinning and tracking it.

## How the Exam Builds Wrong Answers

The wrong answers here describe things careful developers really do: benchmarking, picking a mid-range model, upgrading to the newest release. What makes them wrong is a hidden assumption.

- **A guessed target instead of a real one.** The option names a model or a benchmark while the quality bar, wait time and budget are still blank. A public benchmark has the same problem: it measures someone else's work and sets no bar for yours.
- **Optimising one target and ignoring the other two.** "Maximum capability" or "lowest price" treats the decision as one-dimensional. Ask what the option quietly does to the other two numbers.
- **Treating an upgrade as a string swap.** The option assumes the prompt, settings and token maths all carry over. Some of that breaks with a `400`. Some of it just costs more.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| Only functional requirements, nothing on speed, cost or quality | Get all three targets, then choose | Naming a model or benchmark before the targets exist |
| Cost is the problem, quality is fine | Compare cost per finished task, focusing on the hardest tasks | Comparing per-token price on a typical request |
| Speed is the problem, quality is fine | Something that leaves the output alone: lower effort, streaming, fast mode | A smaller model, which changes the output |
| A working integration moves to a newer model | Re-check settings, prompts and token counts | Assuming a drop-in because the request format still works |
| A production model has a retirement date | Migrate before it, because requests fail after | Treating deprecation as a suggestion, or assuming every platform has the same dates |

One habit works beyond this table: for every option, ask which of the three targets it changes, and which two it leaves undefined.

## What to Remember

- The **three targets** are the quality bar, the latency ceiling and the cost per unit. Get all three before shortlisting models.
- The current lineup, cheapest first: **Haiku 5.5, Sonnet 5.5, Opus 5.5, Fable 5.2**. **Start with Opus 5.5** and let evals move you up or down. Check the lineup when you build, because it changes.
- Compare **cost per finished task**, not per token. Price the **hardest 10%**, because failed tasks bill their tokens, the retry and the downstream damage.
- **Let an eval decide** model changes in both directions. Upgrading to a newer model is often the cheapest win.
- **Opus 5.5 and Fable 5.2 always think.** Disabling thinking returns a `400`; use effort instead. Opus 5.5 and Haiku 5.5 default to `medium` effort.
- **Fast mode buys speed only**: same model, same quality, up to 2.5 times the output speed, at twice the price.
- A **dateless model ID is a fixed snapshot**, not a pointer to the newest model. Older short names are aliases to the latest dated snapshot. The weights stay fixed, but the serving system around them doesn't.
- **Upgrades change behaviour, not just the ID.** Loud breaks are `400` errors (disabled thinking, forced tool choice, sampling parameters, prefill). Quiet costs come from old prompts that newer models follow too literally. **Retired models fail** after at least 60 days' notice.

## Endnotes for this chapter

1. The current lineup with API IDs, descriptions, context windows, maximum output, prices, latency, thinking mode and default effort; the recommendation to start with Claude Opus 5.5 for most workloads and use Claude Fable 5.2 for demanding reasoning, long-horizon agentic work, or when evals on Opus 5.5 at higher effort fall short; Haiku 5.5 as the fastest and cheapest option and Sonnet 5.5 as the balance of speed and intelligence; and the 200K context window and 32K–64K maximum output of Opus 4.5, Sonnet 4.5 and earlier models. Anthropic, *Models overview* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/models/overview](https://platform.claude.com/docs/en/models/overview)

2. Claude Haiku 5.5 pricing by prompt length: $0.10 input and $0.50 output per million tokens for prompts up to 100,000 tokens, and higher prices for prompts over 100,000 tokens, with cache reads and writes counted in prompt length. Anthropic, *Claude Haiku 5.5* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/models/haiku-5-5/overview](https://platform.claude.com/docs/en/models/haiku-5-5/overview)

3. Cost and intelligence levers: the model order from cheapest to most capable; pricing on cost per completed task; the SWE-bench Pro results in which Opus 5.5 at default effort roughly matched Fable 5.1 at its default for about a fifth of the cost, and Opus 5.5 at low effort matched Sonnet 5.5 at its default for about 7% more per solved task; the DeepResearch Bench II result in which Fable 5.2 cost 2.9 times Opus 5.5 for a score 1.9 points lower; the WideSearch run in which 2 of 20 problems carried 43% of spend; the GPQA Diamond result in which Haiku 5.5 scored 85% against 91% for Opus 5.5 at about 1/20 of the cost per correct answer; the Opus 5 to Opus 5.5 upgrade at about the same score and 78% less per solved task; the DRACO result in which low effort cut task time by 84% for a 13.3-point lower score; Haiku 5.5's 5x price above 100,000 prompt tokens and advice to compact before it; fast mode on Fable 5.2 and Opus 5.5 at up to 2.5x output speed and twice the token price; about 30% more tokens for the same text on Claude 4.7 and later; and the support-desk prompt audit in which the same prompt cost more on Opus 5 than Opus 4.8, while the audited prompt was cheaper and more accurate. Anthropic, *Optimizing for cost and intelligence* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)

4. Effort levels `low`, `medium`, `high`, `xhigh` and `max`; a default of `high` on most models and `medium` on Claude Opus 5.5 and Claude Haiku 5.5; `xhigh` absent on Claude Mythos Preview, Opus 4.6, Opus 4.5 and Sonnet 4.6; and `max` absent on Opus 4.5. Anthropic, *Effort* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/effort](https://platform.claude.com/docs/en/build-with-claude/effort)

5. Claude Opus 5.5 breaking changes and behaviour differences: thinking always on, with disabled thinking and manual `budget_tokens` returning a `400`; forced tool use (`tool_choice` of `any` or a named tool) returning a `400`; thinking blocks tied to the model and conversation, with changes before a thinking block able to return a `400`; default effort of `medium` where Opus 5 ran at `high`; more thinking per turn at the same effort; and text between tool calls arriving in `thinking` blocks. Anthropic, *What's new in Claude Opus 5.5* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/models/opus-5-5/whats-new-opus-5-5](https://platform.claude.com/docs/en/models/opus-5-5/whats-new-opus-5-5)

6. Fast mode for Claude Fable 5.2 at up to 2.5x higher output tokens per second, set with `speed: "fast"` and no beta header. Anthropic, *Claude Platform release notes* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/release-notes/overview](https://platform.claude.com/docs/en/release-notes/overview)

7. Model IDs from the 4.6 generation onward use a dateless format, and each dateless ID is a pinned snapshot rather than an evergreen pointer; older models have short aliases that resolve to the most recent dated snapshot; and the serving infrastructure around a model can change and occasionally cause minor behaviour differences on an unchanged ID. Anthropic, *Model IDs and versioning* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions)

8. At least 60 days' notice before retiring a publicly released model, requests after retirement failing, and partner platforms setting their own retirement schedules. Anthropic, *Model deprecations* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/model-deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations)

9. Claude Sonnet 4.5 retiring on the Claude API on November 30, 2026, and Claude Opus 4.5 on January 22, 2027. Anthropic, *Claude Platform release notes* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/release-notes/overview](https://platform.claude.com/docs/en/release-notes/overview)

10. Non-default `temperature`, `top_p` or `top_k` and assistant-message prefill returning a `400` error on Claude Opus 4.7 and later. Anthropic, *Migration guide* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/about-claude/models/migration-guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)
