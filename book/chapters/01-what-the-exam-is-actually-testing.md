# Chapter 1: What This Exam Tests That an Interview Doesn't

**Summary**: *An interview asks whether you can build the thing. This exam asks whether you picked the right mechanism, and almost every wrong answer is a mechanism that works well somewhere else. This chapter sets up that idea with the basics: what a token is, what the context window is, why the same request can return two different answers, and which setting does which job. These four facts make a surprising number of later questions easy.*

## The Same Request Twice Can Return Two Different Answers, and Nothing Is Broken

Send a request. Then send it again, exactly the same, byte for byte. The two responses will often use different wording, sometimes a different structure, and occasionally one includes a caveat the other leaves out.

The natural reaction is to open a bug ticket. Surely something upstream changed: a timestamp in the prompt, a redeployed config, two versions of the code behind a load balancer.

Usually nothing changed. The model **generates one token at a time**, and each token is **sampled** from a set of candidates, not looked up. Two runs can pick different tokens at any point and then continue differently from there. Anthropic says this directly: even with temperature set to 0, results are **not fully deterministic**, and identical inputs can produce different outputs.<sup>[1]</sup>

On current models you can't even set the temperature. Starting with Claude Opus 4.7, setting `temperature`, `top_p` or `top_k` to anything but the default returns a `400` error, and every current model follows that rule.<sup>[2]</sup> So "turn the temperature down" isn't an option any more. Even on older models where it was, it reduced variation but never removed it.

People reading the output don't notice the rewording. A **parser** does, and so does any later step that expected a field where a sentence now sits.

This is where the exam differs from an interview. In an interview, "it varies, that's expected" is a fine answer. On the exam, a scenario will describe the variation, put a believable bug next to it, and ask what you tell the teammate who found it. **The right answer names the cause and then says what the application does about it.** Stopping at "that's expected" explains the behaviour but fixes nothing.

What the application does about it:

- **Validate** the output, and **retry** when validation fails.
- **Constrain the format**, with structured outputs or a strict tool schema (chapter 6).
- **Show examples** in the prompt of exactly the output you want (later in this chapter).

None of these make the model repeat itself word for word. They make the system around it tolerate the fact that it won't.

> **Example.** A teammate reports: "Our classifier returned `"Billing"` yesterday and `"billing issue"` today for the same ticket. Something in the request must have changed."
>
> - **Weak answer:** "Nothing changed. Outputs vary. That's expected."
> - **Strong answer:** "Outputs vary between runs even with identical input, so we shouldn't rely on free text. Let's constrain the output to a fixed list of labels with structured outputs, and validate the label before we use it."

Remember the shape of the strong answer, because it comes back again and again: **a property of the technology, then the part of the system that handles it.** An option that only gives the first half is describing the world. An option that only gives the second half treats a permanent property as a bug with a fix.

## A Token Is What You're Billed In, Limited By and Debugging Against

A **token** is the smallest unit the model works with. It can be a word, part of a word, a character or a byte. On older Claude models, a token is about **3.5 English characters**. Claude 4.7 and later models use a newer tokenizer that produces **about 30% more tokens** for the same text.<sup>[3]</sup> The ratio also changes with language and content type.

This matters because **three different things** on the exam are measured in tokens, and people who haven't absorbed that treat them as three unrelated topics:

| What's counted in tokens | Where it matters |
|---|---|
| Everything you send, on every request | Your bill, and how much of the context window is left |
| Everything the model writes back | Your bill again, at a higher rate, and the output limit |
| Everything the model spends thinking before it answers | Your bill a third time, billed as output, even if you never see the thinking text<sup>[4]</sup> |

The third row surprises people, and it's the reason the **effort** setting exists.

> **Example.** Your prompt is 2,000 tokens. The model thinks for 3,000 tokens and writes a 500-token answer. You're billed for 2,000 input tokens and **3,500 output tokens**, not 500. And `max_tokens` has to be at least 3,500, or the answer gets cut off.

## The Context Window Is Capacity; Sampling Is Choice

The **context window** limits how much material one request can carry. It's finite, and it works like **working memory, not storage**: as it fills up, accuracy and recall get worse. Anthropic calls this **context rot**.<sup>[5]</sup>

Now for the most useful confusion to clear up in this chapter, because scenarios use it again and again, usually voiced by a confident colleague.

**How a token gets chosen has nothing to do with how much text fits.** Sampling decides *which* candidate wins at each step. It doesn't change capacity at all. So if a transcript won't fit and a colleague suggests changing the sampling settings, that isn't a weak fix. It's a fix aimed at the wrong part of the request. The wording of the failure might change, but the failure stays.

Here are the main settings, side by side:

| Setting | What it changes | What it can't change |
|---|---|---|
| Sampling (`temperature` etc., older models only) | Which token gets picked from the candidates | How much input fits, or what the response costs |
| Context window | How much material one request can hold and cross-reference | How varied the wording is |
| `max_tokens` | How long the response, including thinking, can be | How much input fits |
| Effort | How much the model thinks before answering | How much input fits |

Think of these as **four separate machines** on the same request. The mistake also works in reverse: if the wording is **inconsistent**, cutting the input won't fix it.

One rule follows directly and settles a whole class of questions. When a scenario needs a **long document understood as a whole**, so that a reference in section nine makes sense against something in section two, **both sections must be in the same request**. That's a capacity question. No sampling setting, speed improvement or output limit can help.

## Model, Thinking and Effort Are Three Separate Settings

Many developers assume there's one dial for "how good should it be", and that the dial is the model name. In fact there are **three separate settings**, and exam scenarios deliberately pull them apart:

- **Model** sets the capability tier (chapter 4).
- **Thinking** decides whether the model reasons before answering. With **adaptive thinking**, the model decides how much each request needs. On most current models, thinking is on by default and can't be turned off.<sup>[6]</sup>
- **Effort** controls how many tokens the model spends, trading thoroughness for efficiency on the same model. It's available on all supported models with no beta header.<sup>[7]</sup>

A fourth setting is about speed, not quality. **Fast mode** gives up to **2.5 times more output tokens per second** on Claude Opus 5.5 and Claude Fable 5.2, at **twice the normal price**.<sup>[8]</sup>

| Setting | What you get | What it costs |
|---|---|---|
| A bigger model | Better results on genuinely harder work | More money per token, usually more latency |
| Thinking | Reasoning before the answer, for work that needs it | Tokens the user never reads |
| Effort | Control over how much reasoning happens | Thoroughness, when turned down |
| Fast mode | More output tokens per second | Twice the token price, on supported models only |

Two of these are quality settings, one is a speed setting, and they're priced differently. That's what a scenario tests.

There's also a practical trap. **Changing the thinking configuration or the effort level invalidates prompt caching.** Both are written into the prompt itself, so changing them changes the prompt. Anthropic's advice is to treat any thinking or effort change as wiping the whole cache.<sup>[9]</sup>

> **Example.** A team caches a 50,000-token system prompt and reuses it across thousands of requests. Then they add a rule: "for hard questions, set effort to `high`; otherwise `low`." Every time the effort flips, the cache is invalidated and they pay full price for those 50,000 tokens again. The fix is to keep effort steady within a conversation, or use the per-message effort feature, which keeps the cache.<sup>[10]</sup> An interview never asks about this kind of interaction between two features. This exam does.

## Three Ways to Prompt, Priced in Tokens

The last basic idea is the cheapest technique in the book. People often skip it because it seems too simple to be the answer.

- **Zero-shot**: describe the task, give no examples.
- **One-shot**: give one worked example.
- **Multi-shot**: give several examples.

Moving down this list **trades tokens for accuracy**. A description states a rule and leaves the edge cases open to interpretation. An example shows one edge case already decided. When written instructions keep producing the wrong capitalisation, or the wrong handling of an awkward input, one example often fixes what another paragraph of instructions didn't. The cost repeats, because the examples are sent with every request.<sup>[11]</sup>

> **Example.** You want dates written as `2026-11-27`, but the model keeps writing `27 Nov 2026` despite the instruction "use ISO format". Adding one example input and output pair to the prompt usually fixes it immediately:
>
> ```
> Input: "Meeting moved to the 3rd of March"
> Output: {"date": "2027-03-03"}
> ```

Here's the scenario shape to recognise. A team has a **small set of labelled examples** showing exactly the output they want. The options are: put them in the prompt, store them in a database, train a custom model, or warn users away from the hard cases. **Put them in the prompt.** It's the shortest path from what the team already has to the output it wants, and nothing has to be built before you can try it.

**Reaching past the cheap option** is the mistake, and it often looks like rigour. Training a custom model on a handful of examples isn't more serious than showing the model those examples. It's the **same information** sent through a much more expensive channel, with a deployment to maintain afterwards.

## How the Exam Builds Wrong Answers

Wrong answers here are rarely absurd. They describe real mechanisms that a good developer uses every week. What makes one wrong is a **single hidden assumption**, and on this material there are three to remember:

- **A generation setting offered for a capacity problem.** Sampling settings and `max_tokens` affect how the response is *produced*. None of them change how much input fits. When a scenario is about material that doesn't fit, any option about sampling is wrong on mechanism, not just weaker.
- **Expected behaviour diagnosed as a bug.** Different wording on an unchanged request is how generation works. An option that confirms a bug, files a ticket or tries to pin something to stop it has accepted a false premise. Also wrong: an option that explains the cause and then stops, when the question asked what to do.
- **A heavy mechanism chosen over a light one that already fits.** Training, retrieval infrastructure and separate applications all show up as answers to problems that a few examples in the prompt would solve. **Cost is the clue**: ask what each option needs built before you can even try it.

Match on what the scenario is doing, not on the industry it's set in:

| When the scenario says | The answer is usually | The trap is usually |
|---|---|---|
| The same request returns differently worded responses | Name sampling as the cause, then validate, retry or constrain the format | Confirming a bug in the request, or blaming the model version |
| A document is too long, or references across it get lost | Capacity: fit it in one request, or split it and carry state deliberately | A sampling setting, a speed fix, or a bigger output limit |
| A small set of labelled examples already exists | Put them in the prompt as examples | Training a custom model, or storing them where the model never sees them |
| Output quality is uneven on work that needs reasoning | Thinking or effort, chosen for what the tokens buy | A bigger model as the only quality dial |
| Speed is the problem and quality is fine | A speed mechanism, priced accordingly | Anything that changes *what* the model produces |
| A large, stable prefix is reused across many requests | Keep the conditions the cache depends on | Changing thinking or effort per request, which silently breaks the cache |
| A scenario sets `temperature` on a current model | Remove it and steer with prompting or structured outputs | Lowering it to "fix" variation, which now returns a `400` |

One habit is worth more than the table. When two options both look reasonable, ask which mechanism each one acts on, then check which mechanism the scenario is actually about. Here, the wrong option almost always works on a real mechanism the scenario never mentioned.

## What to Remember

- A **token** is the unit of **billing**, of the **context window** and of **thinking**. Input, output and thinking are all counted in tokens. Thinking is billed as output. Claude 4.7 and later models produce about **30% more tokens** for the same text than older ones.
- Each token is **sampled**, so the same request can return different text, **even at temperature 0**. That's how generation works, not a bug. Handle it with **validation, retries, structured outputs and examples**.
- **Current models reject `temperature`, `top_p` and `top_k`** with a `400` error. Steer them with prompting instead.
- **Sampling chooses; the window holds.** A capacity problem never responds to a sampling change, and inconsistent wording never responds to cutting the input.
- The **context window is working memory, not storage**. Recall degrades as it fills (**context rot**). Anything that must be cross-referenced has to be in the **same request**.
- **Model, thinking and effort are three separate settings**, not one quality dial. **Fast mode** is a speed setting: more output per second at twice the price, on Opus 5.5 and Fable 5.2.
- **Changing thinking or effort invalidates the prompt cache**, so switching them per request can quietly cancel the savings a large cached prefix was earning.
- **Zero-shot, one-shot and multi-shot** trade tokens for accuracy. When a team already has labelled examples, **putting them in the prompt** is the cheapest fix, and reaching past it is the usual wrong answer.

## Endnotes for this chapter

1. Temperature controls the randomness of predictions; even with temperature set to 0, results are not fully deterministic and identical inputs can produce different outputs. Anthropic, *Glossary* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/glossary](https://platform.claude.com/docs/en/about-claude/glossary)

2. Starting with Claude Opus 4.7, setting `temperature`, `top_p` or `top_k` to any non-default value returns a 400 error, and the recommended path is to omit them and guide behaviour with prompting; Claude Sonnet 5 introduced the same rule for Sonnet-class models; and in API version 2026-10-21, sending any of the three returns a 400 error even at the default value. Anthropic, *What's new in Claude Opus 4.7* and *What's new in Claude Sonnet 5* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7); Anthropic, *Migrating to API version 2026-10-21* (accessed 2026-11-27), [https://platform.claude.com/docs/en/api/migration-2026-10-21](https://platform.claude.com/docs/en/api/migration-2026-10-21)

3. Claude 4.7 and later models and Claude Mythos Preview use a newer tokenizer that produces approximately 30% more tokens for the same text than earlier models, on which a token represents approximately 3.5 English characters. Anthropic, *Glossary* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/glossary](https://platform.claude.com/docs/en/about-claude/glossary)

4. Thinking tokens are billed as output tokens even when the thinking text isn't returned, and count toward `max_tokens` alongside the response text. Anthropic, *Thinking* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/thinking](https://platform.claude.com/docs/en/build-with-claude/thinking)

5. The context window works as working memory rather than storage, and accuracy and recall degrade as the token count grows, which Anthropic calls context rot. Anthropic, *Context windows* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/context-windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)

6. Thinking runs by default when the thinking parameter is omitted on Claude Opus 5, Sonnet 5.5, Sonnet 5 and Haiku 5.5; thinking can't be disabled on Claude Opus 5.5 or Sonnet 5.5. Anthropic, *Prompting best practices* and *Claude Sonnet 5.5 migration guide* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices); [https://platform.claude.com/docs/en/models/sonnet-5-5/migration-guide](https://platform.claude.com/docs/en/models/sonnet-5-5/migration-guide)

7. The effort parameter controls how many tokens Claude spends when responding and is set in `output_config.effort`, with levels `low` to `max`. Anthropic, *Effort* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/effort](https://platform.claude.com/docs/en/build-with-claude/effort)

8. Setting `speed: "fast"` on Claude Fable 5.2 or Claude Opus 5.5 gives up to 2.5x higher output tokens per second at twice the standard token prices. Anthropic, *Optimizing for cost and intelligence* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)

9. The thinking configuration and resolved effort level are rendered into the prompt, so changing either changes the prompt; switching thinking modes invalidates cache breakpoints, and any thinking or top-level effort change should be treated as invalidating the whole cache. Anthropic, *Thinking* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/thinking](https://platform.claude.com/docs/en/build-with-claude/thinking)

10. Changing the top-level effort between requests invalidates the prompt cache for the conversation's messages, while a per-message effort change, which needs a beta header and adaptive thinking, keeps the cache. Anthropic, *Prompting Claude Haiku 5.5* (Claude Platform Docs, accessed 2026-11-27), [https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-haiku-5-5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-haiku-5-5)

11. Prompting techniques, including worked examples, are documented as a way to improve adherence to a required output format. Anthropic, *Prompt engineering overview* (Claude Platform Docs, accessed 2026-09-09; not re-checked for this edition), [https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview)
