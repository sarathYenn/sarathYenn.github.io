---
layout: ../../layouts/PostLayout.astro
title: "The Class You Didn't Train For"
description: "Classifiers are built to pick a winner from a fixed list. Most on-device recognition problems are the other kind — where the honest answer is usually 'nobody I know' — and the gap between those two framings is where beginners lose months."
date: 2026-08-14
tags: ["edge-ai", "machine-learning", "recognition", "evaluation"]
---

*Part 1 of a ten-part series on the engineering problems that define edge AI.*

Almost every introduction to machine learning teaches the same first task. You have a picture; it is a cat or a dog. You have an email; it is spam or not. You have a handwritten digit; it is one of ten. The model produces a score for each option, the scores are squeezed until they sum to one, and the largest one wins.

This framing is called **closed-set classification**, and the name contains its own assumption: the set of possible answers is closed. Every input you will ever see belongs to one of the categories you trained on. Under that assumption, "which one" is the whole question, and picking the largest score is a complete answer.

Now put a model on a device that lives in the world. A doorbell that should recognize the four people who live in the house. An earbud that should notice when its owner is being spoken to. A watch that should identify the handful of voices its wearer knows. In every one of these, the overwhelming majority of what the sensor picks up belongs to **none of the trained categories**. The delivery driver is not one of the four residents. The voice in the next seat belongs to a stranger. The correct answer, most of the time, is *nobody I know* — and that is precisely the one answer a closed-set classifier is structurally incapable of giving.

This is **open-set recognition**, and it is the default condition of on-device AI rather than an exotic corner of it. Understanding the difference early will save you from a category of failure that is very hard to see and very easy to ship.

## Softmax cannot say "none of the above"

The mechanism is worth being precise about, because the intuition matters more than the formula.

A classifier's final layer produces one raw number per known category. Those numbers are passed through a function that exponentiates each one and divides by their total, which forces the outputs to be positive and to sum to exactly one. That last property is what makes them *feel* like probabilities, and it is exactly what makes them dangerous.

Because they must sum to one, they describe a distribution *over the known categories only*. There is no leftover mass, no residual bucket, no place for the model to put the belief that this input is unlike anything it has seen. Present a picture of a bicycle to a model trained on four faces and it will not report confusion; it will report that this is 94% the second resident. The confidence is not a bug in the weights. It is what the arithmetic was built to produce.

Newcomers usually discover this and reach for the obvious patch: *if the top score is below some cutoff, call it unknown.* This is better than nothing, and it is not enough. Softmax scores are notoriously overconfident, and their magnitude reflects how much one known category out-competed the others — not how well any of them actually matched. An input unlike everything in training can still produce a lopsided distribution, because lopsidedness only requires that one option beat the rest, and something always beats the rest.

## Build a space, not a set of buckets

The framing that actually works is different at the root, and it is the standard approach in face recognition, speaker recognition, and most on-device identity problems.

Instead of training a model to output *which of these N categories*, you train it to output a **vector** — a list of a few hundred numbers, usually called an **embedding**. The training objective is not "assign the right label." It is geometric: two inputs from the same identity should land close together in this vector space, and two inputs from different identities should land far apart. The model is never asked to name anyone. It is asked to arrange the world so that sameness becomes proximity.

The payoff is that recognition and training come apart completely. Once the embedding model exists, you recognize someone by computing their vector, comparing it against a stored collection of vectors — the **gallery** — and asking whether the closest one is *close enough*. Adding a new person means storing one more vector. It does not mean retraining anything, which matters enormously on a device that cannot train. (Part 8 in this series is about that property specifically.)

And crucially, this framing has a natural place for *unknown*. If the nearest gallery entry is not close enough, the answer is nobody. The model is no longer forced to distribute all its belief among the known; there is somewhere else for belief to go.

## The threshold is a product decision wearing a technical costume

That phrase — *close enough* — is now carrying the entire system, and where you set it is not a hyperparameter you tune for accuracy. It is a policy choice about which kind of mistake you would rather make.

There are two, and they trade off against each other directly:

A **false accept** is the system matching a stranger to someone in the gallery. It is confidently wrong about an identity. A **false reject** is the system failing to match someone who genuinely is in the gallery. It is unhelpfully silent about someone it should know.

Move the threshold stricter and false accepts fall while false rejects rise. Move it looser and the reverse. There is no setting that eliminates both, and the honest engineering question is never "what is the best threshold" but "which error does this product survive?"

The answer differs wildly by application, and it is worth internalizing how wildly. For a feature that unlocks something valuable, a false accept is a security breach and a false reject is a mild annoyance — so you set it strict and accept that users occasionally retry. For an assistive feature that quietly suggests a name, a false accept means the user confidently says the wrong name to someone's face, which is *worse than the system having said nothing at all*, while a false reject just returns the user to the status quo. Same mathematics, opposite conclusions, and the threshold is where that conclusion gets written down.

## Why "accuracy" is the wrong number to report

Because the two error types are not interchangeable, a single accuracy figure hides everything that matters. Worse, in open-set problems it is trivially gamed: if 95% of what your device encounters is genuinely unknown, a system that answers "unknown" to absolutely everything scores 95% and is completely useless.

The field settled on a better convention, and adopting it early will make your results interpretable to anyone who has worked in this space.

Report the **true accept rate at a fixed false accept rate** — the fraction of genuine matches you catch, measured at a threshold tuned so that impostors get through at some specified small rate, commonly one in a hundred or one in a thousand. This pins down the axis you care about and reports performance along the other one. Two systems compared this way are actually comparable; two accuracy numbers are not.

Alongside it, plot the whole trade-off curve as the threshold sweeps across its range. The curve tells you something a single operating point cannot: whether the system degrades gracefully or falls off a cliff just past where you set it. A system whose curve is nearly flat near your threshold is one you can ship; a system perched on a steep slope will behave differently in the field than in the lab, because the field will shift your operating point whether you consent or not.

## The failure mode nobody warns you about: gallery growth

Here is the result that surprises almost everyone the first time, and it is the single most useful thing in this article.

Open-set performance degrades as the gallery grows, and it degrades for a reason that has nothing to do with model quality.

Consider a threshold tuned so that any given stranger has, say, a one-in-a-thousand chance of being close enough to any given gallery entry. With ten people enrolled, a stranger gets ten chances to accidentally cross that line. With a thousand people enrolled, they get a thousand. The per-comparison error rate is unchanged — the model has not gotten worse — but the per-encounter error rate has climbed by two orders of magnitude, because you are now running far more comparisons and the system reports a match if *any* of them clears the bar.

The practical consequence is severe and specific: **a system evaluated on a small gallery will look far better than the same system in a user's hands a year later**, because real galleries grow. A benchmark at ten enrolled identities can be genuinely excellent while the shipped product becomes unusable at five hundred. Nothing in the evaluation will have warned you, because the evaluation asked the wrong question.

So ask the right one. Sweep the gallery size — ten, fifty, a hundred, five hundred, a thousand — and plot performance against it. That curve, more than any single accuracy figure, tells you whether you have a product or a demo, because it tells you the system's shelf life in ordinary use.

## Building an evaluation that can actually fail

Everything above collapses into a handful of habits. They are unglamorous, and they are the difference between numbers that predict field behavior and numbers that flatter you.

**Your test set must contain impostors.** If every query in your evaluation belongs to someone in the gallery, you have measured a closed-set problem and learned nothing about the open-set one you are shipping. A substantial fraction of queries — a third is a reasonable starting point — should be identities the system has never enrolled, whose only correct answer is *unknown*.

**Enroll under realistic conditions.** It is tempting to build the gallery from your cleanest samples, because that is what a well-run lab produces. But in the field, enrollment happens in whatever conditions the user happened to be in — brief, noisy, badly lit, off-angle. A gallery built from pristine samples and queried with messy ones will report an accuracy the product can never reach. Match the enrollment conditions to reality, even when it makes your numbers worse, *especially* when it makes your numbers worse.

**Fix the operating point before you look at results.** Decide the acceptable false accept rate from the product's requirements, write it down, and then read off the true accept rate. Choosing the threshold after seeing the outcomes is how a system gets tuned to its test set, and the field will not extend you the same courtesy.

**Report the gallery-size curve, not a point.** Always. It is the fastest way to distinguish a system that scales from one that merely demonstrates.

## The shift in stance

Closed-set classification asks a question with a guaranteed answer: *which of these is it?* The model's job is discrimination among known options, and its confidence describes how cleanly it separated them.

Open-set recognition asks a question that is usually answered *no*: *is this any of these?* The model's job is to describe the world geometrically and then let a threshold — a threshold you chose, for reasons about the product rather than the data — decide what counts as near enough to claim.

The thing to carry forward is that the second question is the one nearly every device in the world is actually being asked. A model that has only ever been evaluated on the first will look ready long before it is, and the gap between those two moments is where a great deal of edge AI work quietly goes wrong.

---

*Next in this series: why a model fast enough on average can still miss every deadline that matters.*
