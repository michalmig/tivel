# Independent plan review — prompt

Two parts. **Send Part 1 alone and let the model answer before sending Part 2.**
Part 1 deliberately withholds the repository so the reviewer forms its own
position first; Part 2 then exposes the actual plan. The delta between the two
is the point of the exercise.

---

## Part 1 — send first

```text
You are reviewing a side project at the planning stage. I want your independent
engineering and product judgment, not validation of anything.

A repository for this project exists and you may have access to it. DO NOT open,
read, search, or clone it yet. Everything you need for this part is below. If you
read the repository now, you will anchor on decisions that were already made and
this exercise loses its value.

## Facts about the person

- Senior full-stack developer, ~10 years professional experience. Strongest in
  .NET/ASP.NET; substantial Angular; also Node.js/NestJS. Comfortable across
  frontend, backend, architecture and tooling.
- Works as a contractor through a software house on client engagements.
- This is a side project built in evenings. Not funded. Budget for tooling is
  roughly EUR 35/month (existing AI subscriptions), with willingness to spend
  about PLN 400 for one month of heavier usage if it accelerates things.
- Stated acceptable outcomes, in his words: zero revenue plus a strong portfolio
  piece and a genuinely useful personal tool is acceptable; USD 100-200 MRR would
  be a meaningful win; anything larger only if the market pulls.
- He rarely opens an IDE. Most of his development happens through terminal-based
  coding agents.
- He has built and maintains personal tooling and a set of AI skills covering his
  own software development lifecycle.

## Facts about the problem he observed

- Reviewing non-trivial, largely AI-generated changesets is slow. A diff is
  enough for small changes; for larger features he has to reconstruct intent,
  entry points, runtime flow, what is indirectly affected, and test coverage.
- He has reviewed features written by other developers using AI agents where the
  author could not answer basic questions about their own code ("it works").
- He used CodeRabbit. His assessment: it detects issues, but after using it he
  still could not describe what a changeset did technically without exploring the
  code himself.

## Facts about what he has already built and measured

- For roughly 10 months he has been building and using AI skills that analyze
  changesets and produce written analyses. Three successive versions. Used on
  three different technology ecosystems, at three different clients, on real
  work.
- His assessment of those outputs: they were comprehensible and did let him
  understand what was happening in unfamiliar code. Prompts of this kind work,
  and a good model handles complex setups.
- Weaknesses he reports in the current output: it is a wall of text; it includes
  diagrams that he rarely looks at; asking follow-up questions means more prompts
  and more context switching; results are non-deterministic and the model
  occasionally states something false with confidence.

## What he wants to build

An application that takes a changeset and lets him understand it interactively,
rather than by reading a generated document: navigate the change, ask questions
in context, see what the explanation refers to, and understand both what the
change does for the business and how it achieves it technically.

His stated criterion for the product succeeding: after using it, he should feel
as if a teammate had just walked him through their changeset.

## Facts about the market, as far as they are known

Existing tools in adjacent space include CodeRabbit, Greptile, Qodo, GitHub
Copilot's review features, Sourcegraph, and various "chat with your codebase"
products. CodeSee built codebase visualization and onboarding walkthroughs and
did not survive as a standalone product. Use your own knowledge here; correct or
extend this list if it is wrong or incomplete.

## What I want from you

Reason from these facts. Do not ask me questions before answering - make your
assumptions explicit instead and answer anyway.

1. What is the single riskiest unvalidated assumption in this project, and how
   would you test it for the least possible effort?
2. If you were building this, what would you build first, and in what order after
   that? Justify the ordering.
3. What would you deliberately not build, and why?
4. What is the strongest argument that this project should not be built at all?
   Make that argument properly, not as a disclaimer.
5. Where is this most likely to fail in a way that is expensive to reverse later?
6. What important consideration is missing from the facts above - something you
   would need to know, or something the person appears not to have thought about?

Rules: distinguish facts from your opinions and from speculation. Say explicitly
when you do not know something or when your knowledge may be out of date. No
praise, no hedging, no summary of what I told you. Be concise and concrete.
```

---

## Part 2 — send after the model has answered Part 1

```text
Now read the repository: PRODUCT.md (the product brief and implementation plan)
and CLAUDE.md (instructions for coding agents working in it). These were written
before your answer and reflect decisions already taken.

Compare them against what you proposed.

1. Where does the plan differ from your own reasoning? For each difference, say
   which position you think is right and why. Do not defer to the plan because it
   exists.
2. Which decisions in the plan are internally inconsistent, contradict each other,
   or contradict its own stated decision filter?
3. Which decisions will be expensive or impossible to reverse, and are any of them
   being made earlier than necessary?
4. What does the plan treat as settled that you think is not settled?
5. What is missing entirely?
6. If you had to delete exactly one thing from the planned scope and add exactly
   one thing, what would they be?

Same rules: facts separated from opinion, explicit uncertainty, no praise. If you
think the plan is right where it differs from your own answer, say so plainly and
say what changed your mind.
```
