---
layout: ../../layouts/PostLayout.astro
title: "A Policy That Isn't a Gate Is a Suggestion"
description: "My agent fleet had a simple model policy: plan on the expensive model, build on the cheap one. An audit showed it running exactly inverted — and nothing had errored, because nothing was checking. What it took to make the policy real."
date: 2026-07-09
tags: ["ai", "agents", "developer-workflow", "governance"]
---

I run coding agents in a two-phase loop: a *planning* agent reads the code and produces an implementation plan, then a separate *build* agent executes that plan. The phases get different models on purpose. Planning is the reasoning-heavy step — it's where a stronger, more expensive model earns its price. Building is comparatively mechanical: the plan already names the files, the functions, the test cases. A cheaper model executes a good plan just fine.

So the policy was one sentence: **plan on the expensive model, build on the cheap one.** I wrote it in the project's instructions. Every agent spawn specified its model explicitly. Simple.

Then I audited the transcripts.

## Exactly backwards

Each agent run leaves a transcript, and each turn in it records which model actually produced it. Counting turns per model for one task told the story: the planning phase ran 32 turns on the *cheap* model, and the build phase ran 131 turns on the *expensive* one. The policy hadn't merely drifted — it had inverted. I was buying the discount model for the one step that needed brains, and paying premium rates for the mechanical step, at four times the volume.

Nothing had errored. Nothing had warned. The plans were a bit shallower and the bills a bit higher, and both of those signals are so noisy that I'd never have caught it without reading the transcripts directly.

The root cause was a resume-semantics detail: spawning a fresh agent honors the model you specify, but *resuming* an existing agent — sending a follow-up message to continue it — inherits the model of whoever sent the message. My orchestrator, which runs on the expensive model, was resuming the plan agent to say "plan approved, now build it." That resume silently flipped the agent onto the orchestrator's model. The spawn-time setting it was ignoring didn't complain about being ignored. Configuration that doesn't survive a process boundary doesn't announce that it didn't survive.

## The first fix was a suggestion

My first fix was to write it down harder. The project instructions gained a paragraph: *never resume a plan agent for the build phase; spawn the build agent fresh, with the model set explicitly and the approved plan embedded in its prompt.* A note went into my persistent memory too. Root cause documented, mechanism explained, remediation specified.

This is the fix everyone reaches for, and it has a familiar failure curve: it works exactly as long as the thing reading the instructions remembers to apply them. Instructions to an agent orchestrator are the same class of artifact as a team wiki page titled "Deployment Best Practices" — earnest, correct, and structurally incapable of stopping anyone. The next time context gets long, or the task gets novel, or an agent spawns an agent, the paragraph is just prose in a file. A policy that lives only in documentation isn't a policy. It's a suggestion with good formatting.

## The real fix was a gate

The actual fix took eleven lines of shell. My tooling supports pre-execution hooks — a script that runs before a tool call and can veto it. The hook inspects every agent spawn, and if the spawn doesn't carry an explicit model, it blocks the call with an error explaining the rule:

> *Agent spawn missing model. Set model: "opus" for Phase-A (plan) or model: "sonnet" for Phase-B (build). Never resume a plan agent for build — spawn a fresh agent.*

The first thing I did after installing it was attempt a spawn without a model. It bounced, error message and all, before any code ran. That bounce is the whole point. The rule is no longer something the orchestrator *remembers* — it's something the environment *enforces*. Forgetting is not an available failure mode, because nothing needs to be remembered. The orchestrator that forgets gets told, immediately, in the one place it can't ignore: the failed call it just made.

There's an honest limitation worth stating: the hook can't see resumes, only spawns — so it enforces "every spawn declares its model" mechanically, while "don't resume across phases" stays behavioral. Which is exactly why the gate isn't the only layer.

## Trust needs a ledger

A gate stops the mistakes it can see. For everything else, the answer is the other old idea: audit trails. Three layers, cheapest first:

**The work self-reports.** Every pull request an agent opens carries a line naming the model that built it and the model that planned it. It costs nothing and puts the claim where the reviewer is already looking.

**The orchestrator keeps records.** When it spawns each phase, it writes the assigned model and the transcript path into the task's metadata. The status board can then flag a violation — planning on the cheap model, building on the expensive one — as a visible warning rather than a forensic discovery.

**The transcript is ground truth.** Self-reports can be wrong; the per-turn model IDs in the transcript cannot. One detail matters when checking: count the models as *consecutive runs*, not as a deduplicated set. A two-phase agent legitimately shows two models in sequence; deduplication hides exactly the plan-to-build transition you're trying to verify.

None of these layers prevents anything. That's the gate's job. The ledgers exist because the gate is narrow, and because the day something slips past it, the difference between a five-minute check and an archaeology project is whether anyone was writing things down.

## Why this matters more for agents than for people

A human engineer assigned the wrong tooling notices. The IDE feels different; the compile times change; they complain in standup. An LLM given the wrong model *is* the wrong model — there's no inner engineer left over to notice the substitution. It just produces slightly different work at a different price, fluently, with total confidence. Every quality signal you might catch it by is statistical and delayed.

That asymmetry generalizes past model selection. Anything you care about in an agent fleet — which model runs which phase, which files a task may touch, whether tests ran before a push, whether anything can write to the main branch — sits in the same trap: the workers will never push back on a misconfiguration, so the configuration has to defend itself. Every rule you'd trust a human to internalize needs to be either mechanically enforced or mechanically audited. The rules that are neither are the ones you'll rediscover in a transcript someday, running exactly backwards.

I [wrote earlier](/posts/parallel-agents-serial-reviews) that the discipline of running parallel agents is placing human judgment at exactly two checkpoints — approving the plan, reviewing the merge — and trusting the silence between them. This is the other half of that bargain. The silence is only trustworthy if it's fenced. Gates make the fence; ledgers prove it held. Everything else is a suggestion.
