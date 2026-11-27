# Chapter 8: Cost Comes From What You Resend, Not What You Ask

**Summary**: *A conversation gets more expensive per exchange as it grows, even though each question stays the same size. Nothing about the workload changed. The history was sent again, and it will be sent again next turn too. Once you read the bill that way, token budgeting, usage tracking and caching stop being three topics. They become one question asked three times: what's being resent, at what price, and what could be read back from cache instead of processed again?*

## Turn Forty Costs More Than Turn One Because It Carries Turn One

Ask a **short question** in a **long conversation**, and you're **not billed** for a short question.

Every request **carries the whole history**: the system prompt, the tool definitions, every earlier exchange and every tool result. A forty-turn task has **sent its first turn forty times**. So total cost grows roughly with the **square of the number of turns**, not in a straight line.<sup>[1]</sup>

This book calls that **carried cost**: what a request pays for material you already sent, and already paid for, on earlier requests. It isn't waste, because the history is doing a job. But it's the part that grows while the questions don't, and on a long run it **dwarfs anything you could trim** from the current turn.

> **Example.** Made-up numbers: a system prompt plus tools is 10,000 tokens, and each turn adds 2,000. Turn 1 sends 12,000 tokens. Turn 40 sends 90,000. Across all 40 turns, you send about 2 million input tokens, even though the questions themselves totalled only about 80,000.

Two things follow:

- A cost estimate based on an **average request** will be wrong for long sessions, unless that average came from real long sessions.
- The **strongest lever** isn't deleting history. It's getting the **same bytes billed at a small fraction** of the price, which is what caching does.

## Input, Output and the Cache Rates Are Different Prices

A **token isn't just a token**. Each kind is charged at a different rate, relative to the model's base input price:<sup>[2]</sup>

| What's charged | Rate | Quantity |
|---|---|---|
| Input | 1× | The whole request, history included |
| Output | 5× on current models | The response, capped by `max_tokens` |
| Cache write, 5 minutes | 1.25× | New content behind your cache breakpoint |
| Cache write, 1 hour | 2× | The same, kept longer |
| Cache read | 0.1× on most models; **0.05×** on Opus 5.5 and Sonnet 5.5; **0.025×** on Fable 5.2 | Content that matched a live cache entry |

> **Example.** On Opus 5.5 ($4 input per million tokens), resending a 100,000-token history costs $0.40 uncached and $0.02 as a cache read. Over 40 turns, that's $16 versus about $0.80, plus the one-time writes.

**Output costs five times input** on every current model. So a model that **writes a lot is expensive**, in a way that trimming the prompt never touches. **Thinking tokens** are billed as output too.

The reason is worth knowing, because it makes the fact easy to remember. The **prompt is processed in one pass**. Generation produces **one token at a time**, and each token needs **its own pass** through the model. You're paying for one pass over the prompt, plus one pass per token produced. So spending follows **how much the system says** far more than how much it's told, and a single "average" rate **under-prices** any design that writes a lot.

Three more facts:

- Cheap reads mean a 5-minute cache write **pays for itself after one hit**, and a 1-hour write after two.<sup>[2]</sup>
- **Discounts stack.** The Batch API's 50% discount applies on top of cache pricing.<sup>[2]</sup><sup>[6]</sup>
- **`max_tokens` is invisible to the model.** Lowering it doesn't make Claude write less. It just cuts off the response that needed the room, and you're **still billed** for what was generated. A **task budget** (beta) is different: Claude sees a running countdown and paces its work to fit.<sup>[1]</sup><sup>[9]</sup>

## An Estimate Needs Five Numbers, and You Can Get All of Them Before Launch

Arguments about cost estimates are rarely about the arithmetic. Usually **someone left a number out** and put a comparison in its place.

| Number | Where it comes from before launch | What happens if it's missing |
|---|---|---|
| Requests per period | The forecast, or the traffic this replaces | A price with nothing to multiply |
| Tokens sent per request | **Counted**, not estimated: the token counting endpoint takes a real request and returns the count, free<sup>[5]</sup> | The biggest term becomes an opinion |
| Tokens returned per request | A sample run over representative inputs | The more expensive side of the bill disappears |
| Rates for the chosen model and features | The published price list | You price someone else's design |
| Share of input served from cache | The part of the prompt you can keep stable | Savings overstated by 10× or more |

Two common mistakes look like diligence:

- **The borrowed baseline**: scaling last year's cost per request, or another team's, which brings in a different token profile and price list.
- **The deferred number**: saying token counts "can't be known until production", which removes the very numbers the bill depends on.

Both fail one question: **what here was actually measured?**

## You Can't Cut What You Haven't Measured

Log three things on every call: **tokens in, tokens out, and which feature made the call**.

Without them, a cost spike only answers "why is the total high?" With them, it answers the useful question: which step, on which kind of request? A flow that looks uniformly expensive usually isn't. Often **one step carries most of the spend**.

**Measure first, then optimise.** Without measurement, every fix is across the board:

- A **uniform cut** reduces features that were never driving the cost.
- A **blanket downgrade** to a cheaper model lowers quality on all traffic, when the spend was concentrated in some of it.

Three details decide whether your numbers are right:<sup>[7]</sup>

- **Parallel tool calls** in one turn share an ID and repeat the same usage figures. **Deduplicate them by ID**; don't add them up.
- Once an agent starts subagents, the plain `usage` field **undercounts**. Use the **per-model breakdown** for the whole tree.
- Totals are **per call**. If a session makes several calls, you add them up yourself.

In Claude Code, `/usage` breaks recent consumption down by **skills, subagents, plugins and MCP servers**, and per-user metrics can be exported over **OpenTelemetry**.<sup>[8]</sup>

## Caching Reuses the History Instead of Deleting It

When a multi-turn app gets expensive, the reflex is to cut: keep only the last two messages, summarise every turn, stop sending tool results. Each one shrinks the bill by **damaging the product**, and summarising adds a **billed call every turn**. **Caching reduces the same cost** without touching the content.<sup>[1]</sup>

**How caching works:**<sup>[3]</sup>

- The prompt is built in a fixed order: **tools, then system, then messages**. A change at one level **invalidates that level and everything after it**.
- Matching is **exact, byte for byte**.
- A **cache breakpoint** (`cache_control`) marks where a reusable stretch ends. You can have **up to four per request**, and cache writes only happen at a breakpoint.
- On a miss, the system **looks back up to 20 positions** for an entry an earlier request wrote. A run of consecutive tool calls or tool results counts as one position.
- The simplest setup is **automatic caching**: one top-level `cache_control` field, and the API places the breakpoint on the last cacheable block and moves it forward as the conversation grows. It uses one of the four breakpoint slots.

This explains the most common failure. Put a breakpoint on a **block that changes every request**, and the cache entry is keyed to content that never repeats. You **pay for every write and never read anything back**.

The **moving prefix** is the same problem one step earlier: a timestamp or request ID placed **before** the stable content. Anthropic measured a **25-token status line** at the start of a system prompt raising one run's cost from **$0.59 to $4.24**, which is **worse than having caching off**.<sup>[1]</sup> Per-request text belongs in the **newest user message**.

> **Example.** A system prompt starts with `Current time: 2026-11-27T10:42:13Z`, followed by 8,000 tokens of instructions. The time changes every request, so the 8,000 tokens after it never match a cached entry. Moving the time to the end of the latest user message makes the 8,000 tokens cacheable again.

Other changes also invalidate part of the cache: changing **tool definitions** invalidates everything; toggling **web search, citations or fast mode** invalidates system and messages; changing **`tool_choice`**, adding or removing **images**, or changing the **thinking** or **effort** settings invalidates at least the messages.<sup>[3]</sup>

**Three conditions** must all hold, or caching does nothing:

| Condition | Why it matters |
|---|---|
| The prefix is identical | One changed word before the breakpoint means full reprocessing |
| It repeats within the cache lifetime | 5 minutes by default, refreshed free on each hit, counted from the **start** of the request |
| It's above the model's minimum length | **512 tokens** on current models (Opus 5.5, Sonnet 5.5, Haiku 5.5, Fable 5.2), up to 4,096 on some older ones. Shorter prompts are processed **without caching and without an error**<sup>[3]</sup> |

The silence in that last row is the trap. **Both cache counters in `usage` read zero**, and that's the only signal. Two tools help: `usage` tells you **whether the cache hit**, and **cache diagnostics** tells you **what changed**, naming the first difference among model, system, tools and messages.<sup>[4]</sup>

One pattern defeats the default lifetime: a loop that **waits for a human between turns**. The person comes back in minutes, not seconds, so the cache entry **expires unused**. That's what the **1-hour cache** is for. It costs twice the base input price to write, and pays for itself the first time it prevents a miss.<sup>[1]</sup>

## How the Exam Builds Wrong Answers

Wrong answers about cost are rarely reckless. They're a **real saving applied to the wrong thing**, which is why each sounds reasonable. Four assumptions produce most of them:

- **Deleting when you could reuse.** Truncating history, summarising every turn, dropping tool results. That's right when the material is finished (chapter 7). It's wrong when a **stable prefix is being resent** and the conversation still needs it.
- **A missing number in the estimate.** Ignoring output, postponing token counts until production, borrowing a per-request figure from elsewhere.
- **A blanket fix for a problem nobody located.** A uniform cut, a switch to the cheapest model, accepting the growth as unavoidable. All assume spend is spread evenly, which nobody measured.
- **Treating a control the model can't see as a budget.** Lowering `max_tokens` changes what survives, not what's spent.

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| Cost per exchange climbs as conversations grow, though requests didn't change | Cache the stable prefix, so resent history is billed at the cache-read rate | Keeping only recent turns, or summarising every turn |
| Nobody knows which feature drives the bill | Log token usage per feature, then decide | A uniform cut, a downgrade, or accepting the growth |
| An estimate is needed before launch | Volume, tokens in, tokens out, rates, expected cache share | Scaling a per-request cost from somewhere else |
| Caching is on, and the cache-read counter is zero | Find what changes before the breakpoint, or check the minimum length | More breakpoints, or a longer lifetime |
| High volume, and nobody waits for any single result | The Batch API, at half price | Treating it as interactive, or saving money by cutting quality |
| Spend must be capped without ruining hard tasks | A task budget the model can see, with a hard limit behind it | Lowering `max_tokens`, which cuts off work and still bills it |
| Conversations pause for several minutes while a person replies | The 1-hour cache | The default 5-minute cache, which expires between turns |

One habit works beyond the table. For any option, ask whether it changes **how many tokens are sent**, **what rate they're charged at**, or **only where the cost shows up**. Reporting alone shrinks nothing, and deleting is only one route to a smaller bill.

## What to Remember

- **Carried cost** is what a request pays again for material sent on earlier requests. Because every turn **resends the whole history**, a long task's cost grows roughly with the **square of its turn count**, while the questions stay the same size.
- **Different tokens, different prices**: input 1×, **output 5×**, cache writes **1.25× (5 minutes)** or **2× (1 hour)**, cache reads **0.1×** on most models, **0.05×** on Opus 5.5 and Sonnet 5.5, **0.025×** on Fable 5.2. Thinking is billed as output. Discounts **stack** with the Batch API's 50%.
- An **estimate needs five numbers**: request volume, tokens sent, tokens returned, rates, and the share from cache. All are **knowable before launch**; the **token counting endpoint** is free.
- **Measure before optimising**: log tokens in, tokens out and the calling feature on every call. **Deduplicate parallel tool calls** by ID, use the **per-model breakdown** for subagents, and add up per-call totals yourself.
- **Caching reuses** the history instead of deleting it. Order is **tools → system → messages**, and a change at one level invalidates everything after it. Up to **four breakpoints**, a **20-position lookback**, and **automatic caching** with one top-level field.
- A breakpoint on **content that changes every request** means paying for writes and **never reading**. Keep timestamps and IDs out of the prefix; put them in the newest user message.
- Caching needs **identical bytes**, **reuse within the lifetime** (5 minutes, refreshed on each hit; 1 hour for human-paced loops), and a prompt **above the minimum** (512 tokens on current models). Falling short is **silent**: both counters read zero. **Cache diagnostics** names what changed.
- A **task budget** is visible to the model and changes what it does. **`max_tokens`** is invisible: lowering it cuts off the response and **still bills it**.

## Endnotes for this chapter

1. Every turn of an agentic task resends the growing conversation, so a 40-turn task sends its first turn 40 times and cost grows roughly with the square of turn count; caching bills each resend at the cache-read rate. Per-request content such as a timestamp placed ahead of the stable prefix makes every request a full cache write: a 25-token status line cost $4.24 per run instead of $0.59, more than with caching off. For loops that wait on humans, the 1-hour cache pays for itself on the first prevented miss. A task budget saves money because the model sees a countdown; `max_tokens` caps a single response invisibly, and truncated turns are still billed. Anthropic, *Optimizing for cost and intelligence* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)

2. Cache pricing per million tokens: Opus 5.5 $4 input, $5 5-minute write, $8 1-hour write, $0.20 cache hit, $20 output; Sonnet 5.5 $2, $2.50, $4, $0.10, $10; Haiku 5.5 (prompts up to 100K tokens) $0.10, $0.125, $0.20, $0.01, $0.50; Fable 5.2 $8, $10, $16, $0.20, $40. Cache hits are 0.05x base input on Opus 5.5 and Sonnet 5.5, 0.025x on Fable 5.2 and Mythos 5.2, and 0.1x on other models. Multipliers stack with other pricing modifiers, including the Batch API discount. Anthropic, *Prompt caching* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/prompt-caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching); and *Pricing* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/about-claude/pricing](https://platform.claude.com/docs/en/about-claude/pricing)

3. Cache order is tools, system, then messages; a change at one level invalidates it and everything after. Up to 4 breakpoints; writes happen only at breakpoints; on a miss the system checks up to 20 positions backward, where a run of consecutive `tool_use` or `tool_result` blocks counts as one position. Automatic caching uses one top-level `cache_control` field, places the breakpoint on the last cacheable block, and uses one of the four slots; it is unavailable on Amazon Bedrock's `bedrock-runtime`. The default lifetime is 5 minutes, refreshed free on each use and measured from the start of the request; 1 hour costs 2x base input for writes. Minimum cacheable length is 512 tokens for Fable 5.2, Opus 5.5, Opus 5, Sonnet 5.5 and Haiku 5.5, 1,024 for Opus 4.8 and Sonnet 5, 2,048 for Opus 4.7, and 4,096 for Opus 4.6, Opus 4.5 and Haiku 4.5; shorter prompts are processed without caching and without an error, with both `cache_creation_input_tokens` and `cache_read_input_tokens` at 0. Changing tool definitions invalidates everything; toggling web search, citations or fast mode invalidates system and messages; changing `tool_choice`, images, thinking configuration or `output_config.effort` invalidates at least messages. Anthropic, *Prompt caching* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/prompt-caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

4. Passing the previous response ID as `diagnostics.previous_message_id` makes the API report the first divergence as `model_changed`, `system_changed`, `tools_changed` or `messages_changed`. Diagnostics answers whether the request changed; `usage.cache_read_input_tokens` answers whether the cache hit. Anthropic, *Cache diagnostics* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/cache-diagnostics](https://platform.claude.com/docs/en/build-with-claude/cache-diagnostics)

5. The token counting endpoint accepts the same inputs as message creation and returns the input token count without running inference; it is free. Anthropic, *Token counting* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/token-counting](https://platform.claude.com/docs/en/build-with-claude/token-counting)

6. The Message Batches API processes requests asynchronously at 50% off, with most batches finishing within an hour and results available within 24 hours. Anthropic, *Batch processing* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/batch-processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)

7. Parallel tool calls in one turn share an ID with identical usage data, so deduplicate by ID. Where an agent spawns subagents, the per-model usage map gives whole-tree accounting, because the plain `usage` field undercounts. Each call returns its own total, and an application making several calls accumulates them itself. Anthropic, *Track cost and usage* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/agent-sdk/cost-tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking)

8. `/usage` shows session token usage and, on a plan, attributes recent usage to skills, subagents, plugins and MCP servers; OpenTelemetry export streams per-user token and cost metrics. Anthropic, *Manage costs effectively* (Claude Code Docs, accessed 2026-09-09; not re-checked for this edition), [https://code.claude.com/docs/en/costs](https://code.claude.com/docs/en/costs)

9. Task budgets (beta header `task-budgets-2026-03-13`) set `output_config.task_budget` with `type: "tokens"`, a `total`, and an optional `remaining`; Claude sees a server-injected countdown and paces its work, but the countdown is visible only to the model, with no remaining-budget field in responses. The budget counts what Claude processes in the current agentic loop, not what the client resends. Anthropic, *Task budgets* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/task-budgets](https://platform.claude.com/docs/en/build-with-claude/task-budgets)
