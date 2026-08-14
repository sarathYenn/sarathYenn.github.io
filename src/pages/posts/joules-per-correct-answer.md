---
layout: ../../layouts/PostLayout.astro
title: "Joules Per Correct Answer"
description: "An always-on system is defined by what it does when nothing is happening — which is almost always. The cascade is how you spend nearly nothing on the empty hours and everything on the moments that matter."
date: 2026-08-11
tags: ["edge-ai", "architecture", "energy", "always-on"]
---

*Part 4 of a ten-part series on the engineering problems that define edge AI.*

Consider a device that should notice something specific — a word, a gesture, a particular sound, a face it knows. It must be watching continuously, because the thing could happen at any moment and a detector that is asleep when it happens has failed completely.

Now count how often the thing actually happens. A few times an hour, perhaps. Sometimes a few times a day. The overwhelming majority of what the sensor delivers is nothing at all: silence, an empty room, a pocket. If you sum the energy such a device spends over a day, nearly all of it went into examining moments that turned out to contain nothing.

That observation is the whole of this article. **An always-on system's cost is dominated by its behavior when nothing is happening**, and the architectural pattern that follows from taking it seriously — the cascade — is probably the single most important structural idea in edge AI.

## The metric that changes the design

Before the pattern, the measurement, because the wrong metric produces the wrong architecture.

The intuitive number is energy per inference. It is easy to measure and it points optimization in a misleading direction, because it treats every inference as equally worth running. The number that actually reflects what the system is for is **energy per useful result**: total energy consumed across a realistic day, divided by the number of correct detections produced.

These two metrics recommend opposite things. Under energy per inference, you make the model cheaper. Under energy per useful result, you notice that most inferences produced nothing and ask a better question: *could most of them not have run at all?*

The second question has far more headroom. Making a model twice as efficient is a hard quarter's work with a bounded payoff. Arranging for it to run one-twentieth as often is an architectural decision, and it is available immediately.

## The pattern

A cascade is a sequence of increasingly expensive, increasingly capable stages, where each stage's job is to reject as much as possible so the next one rarely runs.

A typical arrangement has three tiers. The first is a **gate** — something extremely cheap, often not a neural network at all, running continuously. Its only question is *could something be here?* It might measure signal energy, detect motion, or run a tiny classifier a few kilobytes in size. It is not trying to be right about what is happening; it is trying to be cheap and to never miss.

The second tier is a **real but modest model**, running only on what the gate passed. It answers *is this the kind of thing we care about?* — a compact detector, a small keyword spotter, a lightweight classifier.

The third tier is the **full model**, running only on what tier two passed, and only then permitted to be expensive: the large embedding network, the identification pass against a gallery, the multimodal fusion.

The arithmetic of this is what makes it worth doing, and it is worth stepping through concretely. Imagine the gate costs one unit of energy and passes five percent of what it sees. Tier two costs twenty units and passes twenty percent of *that*. Tier three costs five hundred units. Over a thousand time slices, the gate spends a thousand; tier two runs fifty times for another thousand; tier three runs ten times for five thousand. Total: seven thousand.

Running tier three continuously across the same thousand slices would have cost five hundred thousand. The cascade delivers substantially the same results for roughly one-seventieth of the energy. No model was made faster. The system was simply arranged so the expensive one almost never runs.

## The gate is not a small classifier

The most common mistake in building a cascade is treating the first tier as a scaled-down version of the last. It is a different component with a different objective, and getting this backwards produces a cascade that either drains the battery or misses everything.

The gate's two error types have wildly asymmetric costs.

A **false positive** at the gate — waking the next tier for nothing — costs only the energy of one downstream inference, which is then correctly rejected. Mildly wasteful. Entirely recoverable.

A **false negative** at the gate — failing to pass something real — is unrecoverable. The event is gone. No downstream stage will ever see it, because the gate is the only thing that was running. The system did not get the answer wrong; it never learned there was a question.

So the gate is tuned for **very high recall at whatever precision that costs**. A gate that passes twenty percent of its input while missing almost nothing is doing its job well. A gate tuned for accuracy — balanced between the error types, the way a normal classifier would be — is misconfigured, and the misconfiguration shows up as intermittent, unreproducible misses that look like a model quality problem and are actually an architecture problem.

State this as a design rule and it becomes hard to get wrong: **each tier is tuned for recall except the last, which is tuned for precision.** Everything before the final stage exists to avoid discarding, not to decide.

## What the tiers should not share

A cascade only works if the stages are genuinely different. If tier one is simply a smaller version of tier three, trained on the same data with the same objective, it will tend to make the same mistakes — and the case tier one wrongly rejects is disproportionately likely to be exactly the case tier three would have found hard.

Better cascades use **different evidence at different tiers**. The gate looks at raw signal properties. Tier two looks at coarse features. Tier three looks at learned representations. Each stage sees the problem differently, so their errors are less correlated, and the composition is stronger than any of them.

There is a useful diagnostic here. Take the cases the gate rejects, run the full model on them anyway, and see how many the full model would have caught. If that number is large, the gate is not merely aggressive — it is systematically blind to a region of the input space, and the fix is different evidence rather than a looser threshold.

## The wake-up nobody budgets for

A detail that surprises people building their first always-on system: **transitions cost energy**.

Processors in a low-power state do not resume instantly or freely. Waking involves ramping clocks, restoring state, refilling caches, sometimes re-initializing an accelerator. Depending on the platform, a wake-and-sleep cycle can cost as much as several milliseconds of actual work.

This inverts an intuition. The apparently sensible strategy — wake briefly, do a tiny bit of work, sleep again, many times per second — can consume more energy than staying awake, because you pay the transition cost constantly and never remain in the low-power state long enough to profit from it.

The general principle is **race to idle**: do work in concentrated bursts at full speed, then return to a deep sleep state and stay there. Buffer input and process it in batches rather than continuously. Align periodic work so multiple subsystems wake together instead of taking turns keeping the device alive. A system waking a hundred times a second is not idle; it is busy doing nothing, and the power meter will say so.

## Duty cycle dominates everything

Which leads to the number that overwhelms all others in an always-on system: the fraction of time anything is running at all.

A model that is twice as efficient but runs continuously loses badly to a model that is unoptimized but runs two percent of the time. This ratio is usually not a modelling parameter — it is set by the gate's threshold, the buffering strategy, and how aggressively the system is willing to sleep.

The consequence for how you spend your effort is stark, and it is the practical takeaway of this article. When an always-on system misses its power budget, the instinct is to compress the model further. That is frequently the lowest-yield available move. The higher-yield questions are architectural: *can the gate reject more? can we buffer longer and wake less often? does the expensive tier need to run at all in this state?* Those questions routinely find a factor of ten. Compression, at that point, is often fighting for a factor of two.

## Cascades change what "accuracy" means

One last consequence, easy to miss and expensive to discover late.

A cascade's overall recall is the *product* of the recall of every stage. Three tiers at ninety-five percent recall each yield about eighty-six percent overall — noticeably worse than any individual stage, and the shortfall belongs to no single component. Every tier is performing to spec. The composition is not.

This means **the pipeline must be evaluated end to end**, on realistic input, with realistic event rates. Evaluating each tier in isolation on curated data will systematically overstate the system, because each stage was measured on a distribution that the earlier stages will not actually deliver. Tier three, in production, only ever sees input that already survived two filters — a distribution meaningfully different from the one it was benchmarked on.

The corollary is a specific and useful habit: **evaluate each stage on the output of the real stage before it**, never on a clean sample of the whole input space. It is more work to set up, and it is the difference between a number that predicts field behavior and a number that flatters the design.

## The reframing

Optimizing a model asks: how do I make this cheaper to run?

Designing a cascade asks a better question: how do I arrange for this not to run? The first question has bounded returns and a lot of engineering. The second is nearly free and routinely worth an order of magnitude — because the empty hours, which are almost all the hours, are where an always-on device actually spends its life.

---

*Next in this series: the accelerator that quietly declines to run your model and reports success anyway.*
