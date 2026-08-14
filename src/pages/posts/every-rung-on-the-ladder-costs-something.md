---
layout: ../../layouts/PostLayout.astro
title: "Every Rung on the Ladder Costs Something"
description: "Compression is not a switch you flip at the end. It is a descent through precision, structure, and capacity — and the discipline is measuring what each step takes from you before you take the next one."
date: 2026-08-12
tags: ["edge-ai", "quantization", "model-compression", "optimization"]
---

*Part 3 of a ten-part series on the engineering problems that define edge AI.*

A model trained in a datacenter arrives at a device carrying assumptions that no longer hold. It assumes memory is abundant, that arithmetic is cheap, that power comes from a wall. None of these survive contact with something running off a battery in someone's pocket, and closing that gap is the work called **model compression**.

Newcomers usually meet compression as a single word — *quantization* — attached to a single expectation: the model gets smaller and everything else stays the same. It is more interesting than that. Compression is a ladder of distinct techniques with distinct costs, and the engineering is less about knowing the techniques than about knowing precisely what each rung takes from you, measured on the metric you actually care about.

## Why smaller is faster, and it isn't the arithmetic

The first thing to correct is an intuition about *why* compression helps, because getting this wrong leads to optimizing the wrong thing.

The instinct is that a smaller model does less arithmetic and therefore finishes sooner. That is true and it is usually the minor effect. The dominant cost in most on-device inference is not computing with the numbers; it is **moving them**. Weights live in main memory. Arithmetic units live on the processor. Every weight must travel between them, and on the hardware in phones and wearables, that journey costs substantially more energy and time than the multiply it enables. Modern processors spend a great deal of their time waiting for data to arrive.

This reframes everything downstream. Halving the bits per weight halves the bytes crossing that gap and improves the ratio of useful work to waiting — which is why quantization often delivers speedups larger than the raw arithmetic savings predict. It also explains a result that otherwise looks bizarre: a model with far fewer arithmetic operations can be *slower* than a heavier one, if its access pattern is scattered and cache-hostile while the heavier model's is dense and predictable. Operation counts are a poor proxy for speed. Bytes moved and how they are moved is closer to the truth.

## Rung one: fewer bits per number

Models are usually trained in 32-bit floating point — a format with enormous dynamic range, capable of representing values from the microscopic to the astronomical. Trained networks do not need most of it. Weights in a given layer typically cluster in a narrow band, and the vast precision is spent describing distinctions the network's behavior does not depend on.

**Quantization** exploits that. Rather than storing each weight as a 32-bit float, you find the range the layer's values actually occupy, divide it into a smaller number of evenly spaced steps, and store which step each weight lands on. At 8 bits you have 256 steps and one quarter the memory. At 4 bits you have 16 steps and one eighth.

Two details matter more than the mechanism.

**Where the range is measured from.** Applying one shared range to an entire tensor is simple and lossy, because a single unusually large value stretches the range and wastes most of the steps on territory nothing occupies. Computing a separate range per output channel costs a negligible amount of bookkeeping and usually recovers most of the accuracy lost by the cruder approach. When a quantized model degrades badly, per-tensor scaling is among the first things to check.

**Weights versus activations.** Weights are fixed after training; you can inspect their exact distribution and choose ranges optimally. Activations are computed at runtime and depend on the input, so their range must be *estimated* by running representative data through the model and observing what appears. That representative data is the **calibration set**, and its quality quietly determines the quality of the result. A calibration set drawn from a different distribution than production sets ranges that clip real activations in the field, producing a model that benchmarks well and misbehaves in the world. This is one of the most common and least diagnosed quantization failures.

## When arithmetic after the fact isn't enough

Quantizing an already-trained model — **post-training quantization** — is cheap, takes minutes, and often costs less than a point of accuracy at 8 bits. It is always the right first attempt.

When it is not enough, the reason is usually **outliers**: a small number of activations, in a small number of layers, with values far outside the typical range. They cannot be clipped without destroying information, and accommodating them stretches the range so far that everything else collapses into a handful of steps. Attention-based architectures are notably prone to this, which is why they resisted straightforward quantization long after convolutional networks yielded.

The remedy is **quantization-aware training**: rather than quantizing afterward, you simulate the rounding *during* training, so the network experiences the coarser number system while it still has the ability to adapt. Weights shift into distributions that survive rounding; layers that would have been destroyed learn to route around the damage. It recovers most of the lost accuracy and costs a full retraining cycle, which is why it is the second thing you try and not the first.

## Rung two: removing structure

Quantization keeps every weight and describes each one less precisely. **Pruning** takes the opposite route: keep full precision, remove weights entirely.

The observation behind it is that trained networks are substantially redundant — many weights are near zero and contribute almost nothing. Setting them to zero barely changes behavior, and a mostly-zero tensor can in principle be stored and computed far more cheaply.

*In principle* is doing heavy lifting there, and this is where the field's most common disappointment lives. Zeroing individual weights scattered throughout a tensor — **unstructured pruning** — produces impressive sparsity figures and frequently no speedup at all. General-purpose hardware processes dense blocks of numbers. A tensor that is ninety percent zeros, with the zeros in arbitrary positions, still gets processed as a dense block; you have simply arranged for most of the multiplications to be by zero. Unless the hardware and the runtime specifically support the sparsity pattern you produced, the reported compression is on paper only.

**Structured pruning** removes whole units — entire channels, entire attention heads, entire layers. The result is not a sparse model but a genuinely smaller dense one, and a smaller dense model is faster on every piece of hardware ever built. It sacrifices more accuracy per parameter removed, and it delivers speedups that actually appear on a stopwatch. For edge deployment, structured is nearly always the right choice, and the sparsity percentage is nearly always the wrong metric. Measure wall-clock time on the target device or measure nothing.

## Rung three: training a smaller model to imitate a larger one

The third technique is different in kind. Rather than shrinking an existing model, **distillation** trains a new, smaller one to reproduce the behavior of the large one.

What makes it work better than simply training the small model from scratch is subtle and worth understanding. When a large model classifies an image of a particular dog breed, it does not merely output the right answer; it distributes belief across similar breeds in a pattern that encodes real information about how those categories relate. Training the small model to match that whole distribution transmits far more than a bare label does. The large model has, in effect, discovered structure in the problem, and distillation is how that structure gets handed down.

Distillation is the most expensive rung — it requires the large model, a training pipeline, and unlabeled data in quantity — and the most powerful, because it is the only one that changes the architecture rather than compressing what exists. It also composes freely with the others: distill first, then prune, then quantize.

## The discipline that makes it engineering

The techniques are the easy part. What separates a compression effort that produces a shippable model from one that produces a mystery is the process around them.

**Establish the ceiling first.** Measure the uncompressed model, on your real task metric, on your real data, before touching anything. Every subsequent step is measured as a loss against that number. Compression only ever removes accuracy — so if the *unconstrained* model does not clear your product bar, no compression pipeline will rescue it, and finding that out on day one instead of week six is the single most valuable thing this discipline provides.

**Descend one rung at a time, measuring every rung.** Full precision, then half, then eight bits, then four; then pruning; then distillation. Record accuracy, model size, latency, and energy at every step. Skipping to the aggressive end and discovering the model is broken tells you nothing about which step broke it, and you will spend longer bisecting than you saved.

**Measure the task metric, not the proxy.** Compressed models frequently show almost no change in training loss while a specific downstream behavior falls apart — because loss averages over everything, and the thing that broke is a minority case that mattered disproportionately. If the product depends on distinguishing rare events from common ones, measure exactly that. Aggregate metrics are extremely good at hiding exactly the failure you will hear about from users.

**Measure on the target hardware.** Latency on a workstation predicts latency on a phone poorly and on a wearable not at all. Different memory hierarchy, different accelerators, different thermal behavior. Numbers from anywhere but the real device are estimates at best, and Part 5 of this series is entirely about the surprises hiding in that gap.

## The output is a curve, not a model

The last shift in thinking is the most useful.

The instinct is to treat compression as a search for the answer — the one configuration that is small enough and accurate enough. What you should actually produce is a **frontier**: a set of configurations, each of which is the best available accuracy at its size and latency. Nothing on that frontier is dominated by anything else on it; each represents a different, defensible trade.

The frontier is more valuable than any single point on it, for a reason that has little to do with modelling. Product requirements move. A device tier gets added with half the memory. A feature moves from occasional to always-on and the energy budget collapses. A team with a frontier answers those questions in an afternoon by moving along a curve they already measured. A team with one tuned configuration starts over.

Compression is not a final step applied to a finished model. It is the process of learning the exact shape of the trade-off your problem imposes — and that shape, once you know it, is what lets you make a decision instead of a guess.

---

*Next in this series: why the cheapest model in the pipeline is the one that decides whether the expensive ones ever run.*
