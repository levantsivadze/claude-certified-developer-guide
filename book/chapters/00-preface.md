# Preface

## Software Wrote This

A language model wrote this book: every chapter, every table, every sentence. This edition was also updated and rewritten by a language model, in November 2026, to match the current documentation and to read more simply.

That belongs at the front, because it changes how you should read what follows. **Check things.** Every claim taken from Anthropic's documentation has a numbered endnote with a link to the page it came from. The links are there so you can open them. A book written by software shouldn't ask you to just believe it.

Software is good at the job this book needed: reading a lot of documentation closely, keeping all of it in view, and noticing where two pages disagree. Where that happened, the book says so and names the page it followed.

Software is bad at knowing the limits of what it knows. That's what the endnotes are for, and it's why one rule came before all others: **no invented facts**. No made-up company, customer, incident, benchmark or measurement. When a scenario appears, it's clearly marked as an example. When a real figure appears, it comes from a named source. Some examples use made-up numbers to show the arithmetic, and they say so.

## Who This Is For

You've booked the exam, or you're deciding whether to. You're the only reader this book was written for, and that settled every question about what to include.

The exam has **53 items in 120 minutes**. They're multiple choice and multiple response, and each item tells you how many answers to pick. Scores run from 100 to 1,000, and **720 passes**. The exam follows a published blueprint of **8 domains and 25 skills**, with a weight for each skill. This book is organised around that blueprint, and the coverage map at the back shows which chapter covers each skill.

One rule guided the writing: every section has to **help with something the exam tests**. Not just be interesting, complete or thorough. If a passage wouldn't help you answer a question you'd otherwise miss, it isn't here.

There's plenty of good writing about building with these models, but almost none of it is organised around what you'll be asked. This book is a weekend to read and an evening to skim again. The length is deliberate: **a book you finish beats a book you admire**.

## What the Exam Is Actually Like

Two things about this exam shape the whole book.

**Almost every wrong answer is something that really works.** The options aren't nonsense. They're real features used in the wrong place: a Skill where a server was needed, a context file where a hook was needed, a retry where the failure was permanent. Knowing that a feature exists is worth very little. Knowing what it's *for* is worth almost everything.

> **Example.** A question asks how to make sure Claude Code *never* runs `rm -rf` in a project. One option adds "Never run rm -rf" to `CLAUDE.md`. That really works most of the time: Claude reads the file and usually follows it. But "usually" isn't "never". The correct answer is a hook, which runs as code and blocks the command every time. Both are real features. Only one fits the word "never".

**A third of the exam isn't about Claude.** Domain 2 alone is **33.1%**, and it covers REST, versioning, code review and application boundaries. Domain 4, on evaluation, testing and debugging, is just **2.6%**. That's a thirteen-fold difference. If you plan your study time by what seems interesting, you'll spend it in the wrong places. The coverage map at the back lists every weight so you can plan on purpose.

## How a Chapter Is Built

Every chapter has the same parts in the same order, so you can jump into any one of them.

- **An italic summary**: what decision the chapter covers, and what getting it wrong costs.
- **Headings that state their point.** You can read the headings alone as a summary.
- **Tables** wherever things are being compared.
- **Examples** where an idea is abstract: a worked case, some arithmetic or a short code snippet.
- **Named mistakes, in bold.** Wrong answers are built from believable mistakes. A mistake you can name is one you can spot in an option you've never seen before.
- **How the Exam Builds Wrong Answers**: the assumptions wrong answers rely on, then a table of *what the scenario says*, *what the answer usually is* and *what the trap usually is*. The rows are written as general cues, so they work for scenarios in any industry.
- **What to Remember**: a short closing list, written so each line makes sense with the chapter closed.

## How to Read It

**If you have time**, read in order. The chapters follow the order you'd make these decisions when building something, so each one builds on the last.

**If you're short on time**, read all twenty-three "What to Remember" lists first. Then read the chapters behind any line you couldn't explain on your own.

**If you're fixing one weak area**, find the skill in the coverage map and read the chapter that covers it.

One honest warning: **Anthropic's documentation changes.** Models get replaced, prices move, features ship and pages get reorganised. Everything here was checked on the dates in the endnotes, and some of it will have changed by the time you read it. The reasoning lasts longer than the numbers, which is why most of the pages go to reasoning. When a number matters to your answer, follow the link.

Now go and be wrong cheaply, here, rather than expensively, in the exam room.
