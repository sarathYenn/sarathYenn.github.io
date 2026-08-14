---
layout: ../../layouts/PostLayout.astro
title: "A Score Is Not a Probability"
description: "Combining several weak signals into one confident answer requires that the signals be measured in the same currency. Model outputs almost never are, and calibration is the step most fusion designs skip on the way to a number that means nothing."
date: 2026-08-08
tags: ["edge-ai", "calibration", "sensor-fusion", "probability"]
---

*Part 7 of a ten-part series on the engineering problems that define edge AI.*

Devices are covered in sensors, and it is a natural instinct to think that more of them means more certainty. A system trying to identify something has audio, and location, and time of day, and a history of what has happened recently. Surely combining them beats relying on any one.

Usually yes — but only if the combination is done in a way that respects what each signal actually says. The step that makes that possible is **calibration**, it is almost always skipped, and skipping it produces systems whose confidence numbers are decorative.

## Two things called confidence

Consider a system deciding which of several known things it is currently looking at, with two available signals.

The first is a similarity score from an embedding model — a number between minus one and one, where higher means more alike. Suppose the best match scores 0.71.

The second is a contextual prior. The system knows where it is, and historically, at this location, one particular candidate has appeared in about 40% of past encounters.

Both are "confidence." They are not remotely the same kind of quantity. The second is an actual frequency: out of a hundred past events here, forty were this candidate. The first is a distance in a learned geometric space, whose relationship to any real-world frequency is entirely unknown until someone measures it. Is 0.71 a strong match or a weak one? Nothing in the number answers that. It depends on the model, on the training data, on the population being compared against, and on how much variation this particular input contains.

Combining them arithmetically — averaging, multiplying, weighting — is a category error. It produces a number, and the number does not mean anything, and the system will nevertheless act on it.

## What calibration actually is

A model's output is **calibrated** if it corresponds to real-world frequency: among all cases where the model reports 70% confidence, about 70% should turn out correct.

This sounds like it should be automatic and it is not. Neural networks trained with standard objectives are systematically **overconfident** — they routinely report 95% on populations that are correct 70% of the time. The training objective rewards being right, and being emphatically right is not penalized, so the model learns to be emphatic. Larger and better-fitted models are often worse in this specific respect, which is a genuinely counterintuitive result worth remembering.

The tool for seeing this is a **reliability diagram**, and it takes about twenty minutes to build. Bucket your predictions by reported confidence. For each bucket, compute the fraction that were actually correct. Plot claimed against observed. Perfect calibration is the diagonal. Overconfidence bows below it — the model claims more than it delivers, everywhere.

Almost nobody plots this, and it is the fastest way to learn something true about a model you have been staring at for weeks.

## Fixing it is cheap

The encouraging part: calibration is usually correctable with very little machinery, because the *ranking* is generally fine. The model is good at knowing which candidate is most likely; it is bad at knowing how sure to be. That means you can fix the mapping from score to probability without touching the model.

Fit a small correction function on **held-out data** — data the model did not train on — that maps raw scores to observed frequencies. A single-parameter version, sometimes called temperature scaling, simply softens or sharpens the output distribution and fixes most overconfidence with one number. A more flexible version fits a monotonic curve that can correct non-uniform distortion across the range.

Both are inexpensive to fit, cost nothing at inference, and are among the highest-value-per-hour steps available in an on-device system. The essential constraint is the held-out data: calibrating on training data measures a model's memory rather than its behavior, and produces a correction that makes things worse in the field.

## Fusing where the arithmetic works

Once the signals are calibrated, they can be combined — and the right place to do the combining is worth being deliberate about.

**Early fusion** concatenates raw features from all sources and trains a single model over the combined input. It can learn subtle cross-signal interactions and is the more powerful approach when data is plentiful. It also requires every signal to be present, retrains entirely when any one changes, and fails in ways that are hard to attribute.

**Late fusion** runs a separate model per signal and combines their outputs. It is nearly always the better fit on a device, for reasons that are practical rather than theoretical. Each signal's component can be developed, evaluated, and replaced independently. Missing signals degrade the system rather than breaking it — a considerable virtue when a sensor is unavailable, disabled, or denied permission. Each model can be placed on whichever compute unit suits it, which matters a great deal given Part 5. And when the system is wrong, you can see which component was wrong, which is the difference between debugging and guessing.

The natural arithmetic for late fusion is to work in **log-odds** rather than probabilities. Converting each calibrated probability to the logarithm of its odds turns combination into addition, which is numerically well-behaved, keeps everything in a range that avoids underflow, and makes each signal's contribution directly readable as a term you can inspect.

## The assumption underneath, and when it breaks

Adding log-odds is equivalent to assuming the signals are **conditionally independent** — that once you know the true answer, knowing one signal tells you nothing further about another.

This is frequently false, and the direction of the failure is consistent. Correlated signals that agree produce a combined confidence higher than the evidence supports, because the same underlying information got counted twice. A system fusing four signals that are largely restatements of one another will report near-certainty on the strength of a single piece of evidence.

You do not need to solve this properly to benefit from knowing about it. Two mitigations cover most cases. Fit **per-signal weights** on held-out data rather than adding raw terms — correlated signals will naturally be assigned lower weights, because the fit discovers that they add little once the others are present. And **cap the total confidence** below certainty, so no combination of agreeing signals can produce an assertion the system cannot support.

## Missing signals need a defined answer

On a device, signals disappear routinely. Location is unavailable indoors. A permission is revoked. A sensor is off to save power. The system must keep working.

The clean way to handle this falls directly out of the log-odds formulation: a missing signal contributes **zero** — neutral evidence, no shift in either direction. The other signals combine as usual, and confidence is naturally lower because less evidence was available. Nothing special is required.

What must be avoided is the accidental alternative, where an absent signal is represented by a default value that the fusion treats as *evidence*. A location prior defaulting to zero probability is not "we do not know where we are"; it is "we are certain this is the wrong candidate," and it will actively suppress correct answers. This is a common and quietly devastating bug, and it is much easier to prevent by construction — treating absence as a distinct state rather than a magic number — than to find later.

## Measure the lift, and be willing to cut

The last discipline is the one most likely to change what you build.

For every signal, measure the system's accuracy with it and without it. Not the signal's accuracy alone — its **marginal contribution** to the full system. These are very different quantities, and the gap between them is where most disappointment lives.

A signal can be individually informative and add nothing, because everything it knows was already known by another signal. A signal can look weak alone and contribute substantially, because it is informative exactly where the others are uncertain. Only the marginal measurement distinguishes these, and only the marginal measurement justifies the signal's existence.

The reason to insist on this is that **on a device, every signal has a running cost** that has nothing to do with accuracy. It draws power. It occupies memory. It requires a permission that the user may find intrusive, and that permission prompt has a conversion rate. It expands the amount of personal data the system holds, which expands the obligations that come with holding it — the subject of Part 10.

A signal contributing two points of accuracy for a permission prompt, a continuous power draw, and a new category of personal data is not obviously worth having. It might be. That is a judgement, and it can only be made if someone measured the two points. The measurement is the point: **a fusion design is not finished until every input has justified itself, and inputs that cannot are removed.**

## The reframing

The instinct is that fusion is about combination — gather the signals, blend them, get a better answer.

It is really about **commensurability**. Signals cannot be meaningfully combined until they are expressed in the same units, and the work of establishing those units is calibration. A system that adds uncalibrated scores has not fused anything; it has produced a number with the shape of a probability and none of the properties, and it will act on that number with a confidence it never earned.

The good news is that this is one of the cheapest disciplines in the series. A reliability diagram, a held-out set, a one-parameter correction, and a marginal-contribution table for each signal. A day of work, and every confidence the system reports afterward means something.

---

*Next in this series: how a device learns a new person without training, and what it risks forgetting.*
