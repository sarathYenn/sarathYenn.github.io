---
layout: ../../layouts/PostLayout.astro
title: "Agents in CI Fail Silently"
description: "Wiring an autonomous AI reviewer into a pull-request workflow sounds like a config exercise. It's really a lesson in what breaks when you move an interactive tool into a headless one — and why almost none of it fails loudly."
date: 2026-07-01
tags: ["ai", "ci-cd", "developer-workflow"]
---

The pitch is irresistible. You already run an AI coding assistant at your desk; it reads diffs, runs tests, fixes bugs. So you wire it into continuous integration: a reviewer leaves a comment on a pull request, the assistant wakes up in a fresh cloud runner, reads the whole thread, makes the changes, and pushes them back. The loop closes without a human babysitting it.

Setting it up looks like a config exercise — paste a workflow file, add a secret, done. It is not a config exercise. It's a small but complete lesson in what happens when you take a tool designed to sit next to a human and drop it into a context where there is no human at all. And the thing that makes it genuinely hard isn't the number of steps. It's that nearly every way it can break, breaks *silently* — the job goes green, or dies in milliseconds, and tells you almost nothing about why.

## The interactive assumptions you inherit

An assistant that runs at your desk is built around a person being present. That assumption is baked into it far more deeply than the marketing suggests, and it surfaces in three places the moment you go headless.

The first is **approval**. At your desk, when the agent wants to run a shell command, it can pause and ask you to confirm. That pause is a feature — it's the safety rail. But in a cloud runner there is nobody to say yes. So the agent asks, waits, and the request is denied by default. It tries the next command, gets denied, tries again. From the outside the job looks busy: it runs for minutes, burns tokens, produces a plausible-looking transcript. Inside, it's a machine politely knocking on a door that will never open. Nothing errors. The run simply accomplishes nothing.

The second is **authentication**. Interactive auth flows are forgiving — you paste a token into a prompt, and if it's slightly wrong you get a clear rejection and try again. Move that token into a CI secret and the forgiveness evaporates. A secret that's been truncated, or picked up a stray newline on the way in, doesn't fail with "bad credentials." It fails deep inside the network layer, as a malformed header, with a message that names an internal field number rather than the token. The agent initializes cleanly, makes exactly one call, and dies with zero work done. You stare at a stack that mentions headers when the actual problem is a copy-paste.

The third is **defaults**. A desktop tool can afford to hardcode sensible defaults because a human is there to notice when one goes stale. A wrapper that pins a default model identifier will keep pinning it long after that identifier is retired. Headless, there's no one to notice — the run aborts instantly with an "unknown error" and no tokens consumed, and the true cause (a name that no longer resolves to anything) is nowhere in the message.

Three different subsystems, one shared property: each was designed to degrade gracefully *in front of a person*, and each degrades opaquely without one.

## The workflow you're editing isn't the one that runs

There's a fourth trap, and it's the one that wastes the most time because it violates a reasonable intuition.

You put the automation's definition — the workflow file — on a branch, alongside the code it will operate on. You tweak it, push, trigger it, watch it misbehave, tweak again. Nothing you change takes effect. You are editing a file the system is not reading.

The reason is subtle and worth internalizing, because it generalizes far beyond any one CI provider. When a workflow is triggered by activity on a pull request — a comment, a review — the platform has to decide *which version* of the workflow definition to trust. Trusting the version on the pull-request branch would be a security hole: anyone who opened a PR could rewrite the automation and make it do anything, using the repository's own credentials. So the platform makes the safe choice. For those event types, it reads the workflow from the *default branch* — the trunk everyone has already reviewed — and ignores whatever the PR branch says.

The consequence is counterintuitive but airtight: **you cannot iterate on this kind of automation from a feature branch.** Every change to the definition has to land on trunk before it does anything at all. The tight edit-test loop you're used to is gone, replaced by a slow one that runs through your main branch. Teams that don't know this rule can burn an afternoon changing a file that was never in the loop.

> Imagine a building where the fire drill instructions posted on each floor are ignored, and the only copy that counts is the master binder at the front desk. You can rewrite the sign by the elevator all day; the drill still follows the binder. Editing the right document is half the battle, and it's rarely the document in front of you.

## Green is not the same as correct

Pull these together and a pattern emerges that's bigger than any single fix. Across all four traps, the failure signal is impoverished. A denied-approval loop produces a *successful* run that did nothing. A malformed secret produces an instant death with a misdirecting message. A stale default aborts before doing any work. A workflow read from the wrong branch produces behavior that simply ignores your changes. In none of these cases does the exit code tell you the truth.

That's the real lesson, and it outlasts any particular tool. When you move an agent from interactive to headless, you lose the richest debugging channel you had: a human watching it work in real time, noticing the approval prompt, seeing the wrong model name flash by, catching the auth rejection the moment it happens. Headless, all of that context collapses into a single pass/fail bit and a log you have to go excavate.

So the debugging discipline has to change to match. The instinct to check whether the job "passed" is worse than useless here — passing is one of the failure modes. What you actually have to do is read the *execution trace*: the step-by-step record of what the agent tried, what each tool call returned, where the token counts went to zero. The signal is never in the summary. It's always in the transcript, in a line that says a command required approval, or that a header held an invalid value, or that the model was something that doesn't exist. You have to go and read it.

## What to check, in order

If you're standing this kind of loop up, the distilled version of the hard-won order is short.

- **Prove auth first, in isolation.** Before anything else, confirm the credential actually authenticates from inside the runner. A truncated or newline-poisoned secret is the single most common cause of instant, misdirected death, and it's the cheapest thing to rule out.
- **Grant the agent its permissions explicitly.** In a headless context there is no human to approve commands, so the agent must be handed an allow-list up front. Find out *how* the specific wrapper wants that list — the knob is often not the obviously-named one, and passing it the wrong way fails as silently as not passing it at all.
- **Edit the automation's definition on trunk.** Accept from the start that comment- and review-triggered workflows run from the default branch, and stop trying to iterate on a feature branch. Land the definition on trunk, then test.
- **Pin every default you depend on.** Don't inherit the wrapper's built-in model or version choices. Name them yourself, so a retirement upstream can't silently abort your runs.
- **Read traces, not exit codes.** Treat a green run as unproven until you've seen evidence in the transcript that real work happened — nonzero tokens, actual tool calls, a pushed commit. The summary lies by omission.

---

None of these are exotic. Every one of them is the same underlying shape: a tool that assumed a person was in the room, asked to run in a room with nobody in it. The approval prompt with no one to answer it, the auth flow with no one to retype the token, the stale default with no one to notice, the helpful-but-vague error with no one reading it in real time — they're all the same bug wearing different clothes.

The work of moving an agent into automation isn't writing the workflow. It's re-supplying, ahead of time and in configuration, every judgment a present human would have supplied live. Get that mental model right and the traps stop being surprises and start being a checklist. Get it wrong and you'll spend an afternoon editing a file the system was never reading, watching a green checkmark that means nothing.
