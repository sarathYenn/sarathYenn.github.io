---
layout: ../../layouts/PostLayout.astro
title: "When Brute Force Is the Right Index"
description: "Approximate nearest-neighbour indexes exist to make search over billions of vectors tractable. On a device holding a few thousand, they cost memory, accuracy, and the ability to delete — to solve a problem you do not have."
date: 2026-08-09
tags: ["edge-ai", "vector-search", "embeddings", "performance"]
---

*Part 6 of a ten-part series on the engineering problems that define edge AI.*

Part 1 established the shape of on-device recognition: turn the input into a vector, compare it against a stored collection, and see whether anything is close enough. That comparison step — find the nearest entries in a set of vectors — is one of the most heavily engineered problems in modern infrastructure, with a rich ecosystem of specialized indexes and databases built to solve it.

Nearly all of that engineering was built for a problem you do not have, and adopting it uncritically on a device makes things worse in several directions at once. This article is about how to tell the difference.

## What the comparison actually costs

Start with arithmetic, because the arithmetic settles most of the argument.

Comparing two vectors for similarity means multiplying them element-wise and summing — one multiply-add per dimension. With 256-dimensional embeddings, one comparison is 256 multiply-adds. Comparing a query against a gallery of a thousand entries is 256,000. Against ten thousand, 2.56 million.

Those numbers sound large in isolation and are not. A phone's CPU performs billions of such operations per second, and this particular pattern — a long sequence of independent multiply-adds over contiguous memory — is close to the ideal case for the vector instructions every modern processor provides. Written straightforwardly against those instructions, a ten-thousand-entry search completes in well under a millisecond.

Meanwhile the model that produced the query vector took tens of milliseconds. **The search is not the bottleneck. It is a rounding error on the bottleneck.** Any effort spent optimizing it is effort not spent on the part that actually costs.

It is also worth noting that this operation is memory-bandwidth-bound rather than arithmetic-bound: the processor can multiply faster than memory can supply the vectors. Which means the effective way to speed it up, if you ever genuinely need to, is to make the vectors *smaller* rather than to compare fewer of them — a point that becomes important shortly.

## What an approximate index buys, and from whom

The specialized indexes — graph-based structures, clustered partitions, and their relatives — exist because at a hundred million vectors the arithmetic above stops being free. Twenty-five billion multiply-adds per query is a real cost, and at that scale it is worth accepting approximation to avoid it.

The mechanism is always some form of *not looking at everything*. Cluster the vectors and search only the nearest few clusters. Build a navigable graph and walk it toward the query. Both work by pruning the search space, and both share the defining property: **they can miss.** The true nearest neighbour may sit in a cluster you skipped or off the path you walked. Index quality is measured by *recall* — the fraction of true nearest neighbours actually found — and it is never one.

That approximation is an excellent trade at a hundred million vectors, where the alternative is unaffordable. At ten thousand vectors, where the alternative takes a fraction of a millisecond, you are accepting misses in exchange for nothing.

And the misses are not evenly distributed. Approximate indexes are least reliable precisely at the boundary cases — entries that sit at similar distances from the query, which is exactly the ambiguous situation where correctness matters most. Recall against a random query may be 98%; recall on the hard queries, the ones where the answer is genuinely close, is lower. You have introduced an error source that concentrates on the cases you were most worried about.

## The costs that do not show up in a benchmark

Beyond accuracy, an index on a device brings three practical problems that the server-side literature rarely emphasizes, because on a server they barely matter.

**Memory overhead.** Graph-based indexes store neighbour lists per entry, which can approach or exceed the size of the vectors themselves. On a server, doubling the memory for a large speedup is obviously correct. On a device with a hard memory ceiling and an operating system willing to terminate your process for exceeding it, doubling the footprint to accelerate something that was already imperceptible is obviously wrong.

**Insertion.** On-device galleries are not built once and queried forever; they grow one entry at a time as the user enrols someone new. Many high-performance indexes are built as a batch operation over a complete dataset, and incremental insertion is either unsupported, or degrades structural quality over time, or requires periodic full rebuilds. A rebuild on a phone is an expensive, thermally noticeable event happening at an unpredictable moment.

**Deletion.** This is the one that most often turns into a real problem rather than a performance annoyance. Users delete things, and increasingly they have a legal right to have deletions actually take effect. Many indexes implement deletion as *tombstoning* — marking an entry inactive and excluding it from results while leaving the data in the structure, with true removal deferred to a rebuild. That is fine as a performance strategy and unacceptable as a privacy guarantee. "The record still exists but we filter it out of query results" is not deletion, and if the data in question is personal, the gap between those two things is exactly where a compliance problem lives. With a flat array, deletion is removing an element. There is nothing to explain and nothing to defer.

## Make the vectors smaller instead

If a gallery does grow to where linear search genuinely costs something, the first move is not an index. It is **reducing the size of each vector**, because the operation is bandwidth-bound and smaller vectors mean proportionally less memory traffic.

Two approaches cover most needs.

**Store the embeddings at lower precision.** Embeddings are typically produced as 32-bit floats and rarely need that. Quantizing each dimension to 8 bits quarters the memory and roughly quarters the bandwidth, and the effect on ranking is usually negligible — because you are comparing *relative* distances, and modest uniform noise shifts all of them nearly equally. This is the same idea as Part 3, applied to the gallery rather than the weights, and it is unusually cheap here.

**Reduce the dimensionality.** A 512-dimensional embedding may carry most of its discriminative information in far fewer dimensions. A learned projection, fitted once offline and applied to every vector, can often halve the dimension at a small and *measurable* accuracy cost. Measurable is the key word: unlike an approximate index, whose error depends on the query distribution in ways that are hard to characterize, a fixed projection's cost can be evaluated directly on your task and reported as a single number.

Both approaches shrink the work while preserving exactness of the search itself. You still compare against everything; you simply compare more cheaply. Deletion remains trivial, insertion remains trivial, and there is no recall parameter to tune or degrade.

## Where the crossover actually is

The honest answer is that it depends on dimensionality, hardware, and latency budget — and that you should measure it rather than inherit a number from an article, including this one.

The measurement is straightforward and takes an afternoon. Implement linear search properly, using the platform's vector instructions and a contiguous layout. Generate synthetic galleries at increasing sizes. Measure query latency on the actual target device at each size. Plot it against the latency budget you established in Part 2.

What that curve usually shows, on current mobile hardware with typical embedding sizes, is that linear search remains comfortably inside any interactive budget well past the point where realistic personal galleries stop growing. Which means the crossover is not merely far away — for many on-device applications it is **past the maximum size the gallery will ever reach**, and the index would never have been needed at any point in the product's life.

That is a genuinely useful thing to know early, and it is knowable in an afternoon.

## Layout is where the real speed is

If linear search does need to be faster, the wins are in memory layout rather than algorithms, and they are usually larger than people expect.

Store the gallery as **one contiguous block** — all vectors packed end to end in a single allocation — rather than as a collection of separately allocated objects. The processor prefetches sequential memory aggressively and handles pointer-chasing poorly, and the difference between those two layouts on the same arithmetic can be several-fold.

**Normalize the vectors once, at insertion.** If every stored vector has unit length, cosine similarity reduces to a plain dot product and the per-query normalization disappears entirely.

**Keep only what you need.** Finding the top two matches does not require sorting the results; it requires two variables updated during the scan. Sorting ten thousand scores to read the first two is a surprisingly common and entirely avoidable waste.

None of this is sophisticated. All of it is more effective, on this problem at this scale, than adopting a data structure designed for a fundamentally different one.

## The general lesson

The specialized index is not wrong. It is an excellent solution to the problem of searching hundreds of millions of vectors under a tight latency budget, and the engineering behind it is genuinely impressive.

But the reason it exists is scale, and scale is exactly the thing an on-device gallery does not have. What a device has instead is a hard memory ceiling, incremental writes, a real deletion requirement, and a search cost that was never the bottleneck. Every one of those points away from the index and toward the array.

The habit worth taking from this — and it generalizes well past vector search — is to ask what problem a piece of infrastructure was built to solve, and then to check honestly whether you have that problem. A tool adopted for its reputation rather than its fit brings all of its costs and none of its benefits, and on a device the costs are the ones you can least afford.

---

*Next in this series: why a similarity score is not a probability, and what breaks when you treat it as one.*
