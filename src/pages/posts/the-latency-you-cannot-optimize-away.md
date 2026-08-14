---
layout: ../../layouts/PostLayout.astro
title: "The Latency You Cannot Optimize Away"
description: "Streaming inference has two kinds of delay: the time your model spends computing, and the time it spends waiting for input it has not received yet. Only one of them responds to a faster chip."
date: 2026-08-13
tags: ["edge-ai", "streaming", "latency", "real-time"]
---

*Part 2 of a ten-part series on the engineering problems that define edge AI.*

A model that runs in eighty milliseconds sounds fast. Whether it is fast enough depends on a question that benchmark rarely answers: eighty milliseconds after *what*?

This is the gap between how models are usually measured and how they actually behave in a live system. Offline evaluation hands the model a complete input — a whole image, a whole audio file, a whole sentence — and times the forward pass. On a device responding to the world, the input does not arrive all at once. It arrives continuously, a few milliseconds at a time, and the model must produce answers while the input is still being written. That shift, from *processing a thing* to *processing a stream*, changes what latency means and introduces a component of delay that no amount of hardware will remove.

## The budget is the whole path, not the model

The first correction is to stop thinking about inference time and start thinking about the interval a user can actually perceive: from the moment the world does something to the moment the device responds to it.

Written out, that path has more segments than most people expect. The sensor samples and hands over data in fixed-size blocks, which means the very first thing that happens is waiting for a block to fill. The block gets converted into whatever representation the model expects. The model runs. Its raw output gets turned into a decision — smoothed, thresholded, debounced. The decision reaches the interface, which has its own render or wake cost.

Inference is one term in that sum, and frequently not the largest. Teams that optimize the model in isolation routinely halve the forward pass and find the end-to-end response unchanged, because the real time was going somewhere they were not measuring. The discipline that prevents this is simple and rarely followed: before optimizing anything, instrument the full path and write down where the milliseconds actually go. The answer is usually surprising, and it is almost always more useful than the profile of the model alone.

## Algorithmic latency: the delay a faster chip cannot touch

Here is the idea that separates people who have built streaming systems from people who have not.

Suppose a model needs to see three hundred milliseconds of signal to make a reliable judgement — because that is how long the relevant pattern takes to unfold, and less context makes the decision unreliable. Suppose further that the model runs in one millisecond.

How long after the pattern begins can this system respond? Three hundred and one milliseconds. And if you optimize the model to be a thousand times faster, the answer becomes three hundred milliseconds. You have removed effectively nothing, because the delay was never in the computation. It was in **waiting for input that had not happened yet**.

This is **algorithmic latency**, sometimes called lookahead. It is the amount of future the model requires before it will commit to a judgement about the present. It is determined by the model's architecture and its receptive field, not by the hardware, and it sets a floor beneath your response time that no engineering effort below the architecture level can lower.

Once you see this, a lot of otherwise-baffling results become obvious. Two models with identical benchmark throughput can feel completely different in a live system, because one needs a quarter-second of lookahead and the other needs none. A model quantized and accelerated to run four times faster can leave the perceived response time nearly unchanged. And a system that misses its deadline may need an architectural change — a causal model that only looks backward, a smaller context window, a design that commits early and revises — rather than an optimization pass.

The practical rule: **compute latency and algorithmic latency are separate line items in your budget, and you must know both numbers.** Most teams can recite the first and have never calculated the second.

## Chunks, overlap, and the cost of pretending

Since input arrives continuously and models want fixed-size inputs, streaming systems process in **chunks** — take a window of recent signal, run it, slide forward, repeat.

The chunk size is a direct trade. Small chunks mean less waiting and lower latency, but each inference sees less context and the per-chunk fixed costs — setup, memory transfer, scheduling — are paid more often, so efficiency drops. Large chunks are more efficient and better-informed but respond later. There is no universally right answer; there is only the smallest chunk that still clears your accuracy bar, and finding it is an experiment rather than a guess.

Naive chunking also introduces an artifact worth knowing about in advance: a pattern that straddles a boundary gets cut in half, and neither half looks like the whole. Systems handle this by **overlapping** consecutive windows, so every moment appears in more than one chunk and no event falls into a seam. Overlap costs compute — with fifty percent overlap you process everything twice — which is a real price paid to avoid a real failure.

The more elegant approach, where the architecture permits it, is to make the model **stateful**: rather than re-reading a window of history each time, the model carries a compact summary of the past forward and updates it with each new arrival. This removes the redundant computation entirely and lets context extend far beyond any window. It also introduces the characteristic bug of stateful streaming systems — state that drifts, or fails to reset between sessions, producing errors that depend on everything that happened before and are therefore nearly impossible to reproduce from a single input.

## The tail is the product

Now the measurement mistake that causes the most field surprise.

Average latency is close to meaningless for interactive systems. What users perceive is not the mean; it is the worst of the recent samples. A system averaging forty milliseconds that occasionally spikes to six hundred does not feel like a forty-millisecond system. It feels broken, intermittently, in a way that is hard to describe in a bug report.

So report **percentiles**. The median tells you the typical case. The ninety-fifth and ninety-ninth tell you what a user meets several times a minute in continuous operation — and in a system doing inference many times per second, the ninety-ninth percentile is not a rare event. It is something that happens constantly.

The spikes usually come from a small and predictable set of causes: garbage collection or memory allocation on a path that should not allocate at all, competition with other applications for the same accelerator, the operating system migrating your thread to a slower efficiency core, thermal throttling as the device warms (the subject of Part 9), or a first-run cost — model load, graph compilation, cache warming — being quietly averaged into steady-state numbers where it does not belong.

Two habits address most of this. **Warm up before measuring**, so initialization costs are attributed honestly rather than smeared across the run. And **pre-allocate every buffer** the streaming path uses, so the steady state allocates nothing. A real-time loop that allocates is a real-time loop with a random spike generator wired into it.

## Steady beats fast

A related and slightly counterintuitive principle: for anything a person experiences continuously, **consistency is worth more than speed**.

A system that responds in ninety milliseconds every single time reads as solid. A system that averages fifty but ranges from twenty to four hundred reads as unreliable, even though it is faster on average. The variance — the jitter — is what people notice, because human perception is far better at detecting change than absolute magnitude.

This has a design consequence that feels wasteful and is not: it is often correct to deliberately hold back a fast result to keep the output cadence regular, in the same way video playback buffers rather than displaying frames the instant they decode. Predictability is a feature, and it is one you can only deliver if you are measuring distributions rather than means.

## Spending the budget where it buys the most

When the budget genuinely does not fit, the useful moves are mostly structural rather than numerical.

**Let easy cases exit early.** Most inputs are not hard. A system that reaches a confident answer partway through and stops spends its full budget only on the genuinely ambiguous minority. This idea generalizes into the cascade architecture, which is Part 4 of this series and is arguably the most important structural pattern in edge AI.

**Emit provisional answers and refine them.** Rather than waiting for enough context to be certain, produce a preliminary result quickly and correct it as more input arrives. This is why live captioning appears to change its mind mid-sentence — it committed early, then revised. The interface has to be designed to tolerate revision, but when it can be, this converts a hard latency limit into a soft one.

**Overlap the stages.** While the model processes chunk N, the sensor is filling chunk N+1 and the post-processor is finishing chunk N−1. A properly pipelined system's throughput is set by its slowest stage rather than the sum of all of them, which is often the difference between fitting the budget and missing it.

**Reconsider the architecture.** If algorithmic latency alone exceeds the budget, no amount of the above will help. That is a signal to change the model — to something causal, or shorter-context, or structured to commit early — and recognizing it quickly saves weeks spent optimizing a path that was never going to arrive.

## The reframing

Offline, latency is a property of the model, and making the model faster makes the system faster.

Streaming, latency is a property of the whole path — and part of it is a debt the architecture owes to the future, payable only by waiting. Optimization addresses one part of the budget. The other part is a design decision that was made when the model was chosen, and the earlier you know which part is dominating, the less time you spend making something faster that was never slow.

---

*Next in this series: what you actually give up on each rung of the compression ladder.*
