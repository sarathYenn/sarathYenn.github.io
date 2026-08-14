---
layout: ../../layouts/PostLayout.astro
title: "Learning Without Training"
description: "A device cannot retrain a neural network, and does not need to. Separating the model that learns representations from the memory that stores identities is what lets a system acquire someone new in thirty seconds — and what makes the quality of those thirty seconds decisive."
date: 2026-08-07
tags: ["edge-ai", "few-shot-learning", "enrollment", "continual-learning"]
---

*Part 8 of a ten-part series on the engineering problems that define edge AI.*

A recognition system that only ever knows what it knew at the factory is not much use. Users have their own people, their own objects, their own voices, and the system must acquire them after it ships — on the device, from very little data, without sending anything anywhere.

The instinct is that this requires training on the device, and training on the device is largely impossible. Backpropagation needs the full computational graph in memory, many passes over the data, and an amount of energy that would visibly drain a battery. Nothing about a phone or a wearable makes this reasonable.

Fortunately the instinct is wrong, and the reason it is wrong is the most elegant idea in applied recognition.

## Two things that both get called learning

The resolution is to notice that "the system learns a new person" conflates two entirely separable operations.

The first is learning **what makes people different in general** — which acoustic properties distinguish one voice from another, which facial geometry distinguishes one face from another, and, crucially, which variations do *not* matter: lighting, background noise, mood, a cold. This is genuinely hard, requires enormous amounts of data, and takes serious compute. It is done once, in advance, by whoever trains the model.

The second is learning **that this particular individual exists**. Given a model that already knows how to measure difference, this is just recording a point in the space it built.

The first is training. The second is **enrollment**, and enrollment is cheap. Compute the embedding, store it, done. Milliseconds. No gradients, no optimizer, no energy problem.

This is why the embedding formulation from Part 1 is doing more work than it first appears. It does not merely permit an *unknown* answer. It relocates all the difficulty into a phase that happens before the device ever ships, leaving the device with an operation that is arithmetic rather than learning. The model never changes. The memory does.

## Everything now rests on the enrollment sample

The consequence of that trade is that the entire quality of the system, for any given individual, is determined by the sample used to enrol them. There is no training to compensate. Whatever the embedding captured is what the system will forever believe that person is.

This makes enrollment quality the highest-leverage variable in the whole system, and it is routinely treated as an afterthought.

The trap is specific and easy to fall into. Enrollment is often designed as a controlled moment — hold still, speak this phrase, good lighting, quiet room — because that produces a clean sample and a clean sample seems obviously better. And in the lab it *is* better; the numbers improve.

Then recognition happens somewhere else entirely: across a room, in noise, at an angle, in bad light. The stored representation was formed under conditions that never recur, and the system underperforms in a way that lab evaluation cannot detect, because the lab evaluated queries against a gallery built in the same conditions as the queries.

The principle to hold onto is that **the enrollment sample should resemble the query conditions, not the ideal conditions**. If recognition happens in noise, enrolling in noise produces a better system — even though every offline metric will look worse. This is one of the places where the honest number and the flattering number diverge most sharply, and where trusting the flattering one costs the most.

## Several samples, and a prototype

The natural improvement over one sample is several, combined.

Capture an individual a handful of times, ideally under varying conditions, and average their embeddings into a **prototype**. Averaging suppresses the variation specific to any single capture — that particular background noise, that particular angle — while reinforcing what is consistent across all of them, which is precisely the identity. Three to five samples typically deliver most of the available benefit; the returns flatten quickly after that.

Two refinements matter in practice.

**Normalize before averaging.** If embeddings have unit length, the average of several should be renormalized, or samples with larger magnitude will dominate for reasons that have nothing to do with quality.

**Gate the samples on quality before accepting them.** A sample dominated by noise, or captured with the subject barely present, drags the prototype toward a point that represents nobody. A cheap quality check — signal level, detection confidence, agreement with the samples already collected — that rejects bad captures before they contaminate the prototype is worth substantially more than it costs. The last of those checks is particularly useful: a candidate sample that sits far from the existing prototype is either a valuable new condition or a different individual entirely, and it is worth being suspicious about which.

## Drift, and updating carefully

People change. Voices age or get hoarse. Appearances shift with seasons and haircuts. Devices are replaced with different microphones and different cameras. A prototype fixed at enrollment slowly stops describing the person it represents — **drift**.

The remedy is to update prototypes over time using confident matches: when the system recognizes someone with high certainty, blend that new embedding into their stored prototype with a small weight. The prototype tracks the person, adapting continuously without any explicit re-enrollment.

This works, and it introduces a failure mode severe enough to deserve its own warning.

**The system is now training on its own outputs.** If a confident match is wrong, the incorrect embedding gets blended into the prototype, moving it slightly toward the wrong person. That drift makes the same error marginally more likely next time. Repeated, a prototype can migrate away from the person it was created for and toward someone it was repeatedly confused with — and the process is self-reinforcing, invisible, and produces a system that degrades slowly for reasons no log will explain.

The guards against it are unglamorous and effective. Set the update threshold **much higher than the recognition threshold**, so only near-certain matches are permitted to modify memory — recognizing on weak evidence is fine, learning on weak evidence is not. Keep the blend weight small, so no single update can move a prototype far. **Retain the original enrollment embedding permanently and immutably**, so the prototype can never drift arbitrarily far from a known-good anchor, and so there is something to reset to. And expose a reset: when a user says the system keeps getting someone wrong, the ability to discard an accumulated prototype and re-enrol is often the entire fix.

## What forgetting means here, and where it actually bites

Continual learning literature is preoccupied with **catastrophic forgetting**: a neural network trained on new data overwrites the weights encoding what it learned before, and old capabilities collapse.

The architecture described here is largely immune, and it is worth being clear about why. Each identity is a separate stored vector. Adding one does not modify any other. There are no shared weights to overwrite, because nothing is being trained. Immunity to catastrophic forgetting is not an accident of this design — it is one of its principal benefits, and a strong reason to prefer it on a device even where on-device training is technically feasible.

But a related problem does apply, and it is the one from Part 1 wearing different clothes. As the gallery grows, prototypes crowd the space. Two individuals who genuinely sound or look similar produce nearby vectors, and at some density the threshold that cleanly separated ten identities cannot cleanly separate five hundred. Nothing was forgotten. The space simply ran out of room at the resolution the embedding provides.

This is a property of the embedding model rather than the memory, and it cannot be fixed by better enrollment. It sets a real ceiling on gallery size, and the only way to know where that ceiling sits is to measure performance against gallery size directly — which is why that sweep keeps recurring in this series. It is the measurement that reveals limits nothing else exposes.

## When you genuinely do need on-device adaptation

Occasionally the embedding model itself is wrong for a user — a population, an accent, an environment poorly represented in training. No amount of enrollment fixes a representation that fails to separate the relevant individuals in the first place.

Full retraining remains impractical, but there is a middle path: adapt a **small, isolated part** of the system while leaving the expensive representation frozen. Train a compact transformation that sits after the embedding and reshapes the space to better separate this particular user's set of identities. It is a small number of parameters, trainable from limited data, and cheap enough to fit on a device.

Because it is separate from the base model, this also sidesteps catastrophic forgetting structurally: the general-purpose representation cannot be damaged, since it is never touched. The adaptation is additive and, importantly, discardable — if it makes things worse, remove it and the system returns exactly to its shipped behavior.

## The reframing

The framing that causes trouble is *the device learns*, because learning implies training and training implies impossibility.

The framing that works is that **the device remembers, and the remembering was made cheap by learning that happened elsewhere**. The hard, expensive, data-hungry work of understanding what makes individuals distinguishable was done once, in advance, by someone with a datacenter. What remains on the device is storage, arithmetic, and judgement about which samples deserve to be stored.

That judgement — enrol under realistic conditions, gate the samples, update only on near-certainty, keep an immutable anchor — is where the engineering actually lives. It is not glamorous, and it determines whether the system knows the people it claims to know.

---

*Next in this series: why a benchmark that finishes in thirty seconds tells you almost nothing about a device that runs all day.*
