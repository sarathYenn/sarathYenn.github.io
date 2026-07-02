---
layout: ../../layouts/PostLayout.astro
title: "Parallel Agents, Serial Reviews"
description: "Running more AI coding agents at once looks like turning a throughput dial. It's really the oldest problem in software management wearing new clothes — and past a handful of agents, the dial starts running backward."
date: 2026-07-02
tags: ["ai", "developer-workflow", "parallelism"]
---

You get one AI coding agent working the way you want. It reads a task, writes the change, runs the tests, opens a pull request. It works, and the moment it works, an obvious thought arrives: if one agent can produce a pull request, ten can produce ten. Throughput looks like a dial you can turn. Open more terminals, hand each a task, and multiply.

The dial is real, but it's shorter than it looks, and past a certain point it runs backward. Running many agents in parallel feels like a compute-scaling problem — more workers, more output. It is almost entirely not that. It's an old problem, one software teams have been losing to for fifty years, and the agents don't dissolve it. They just make it arrive faster.

## Isolation is the easy part

The first fear everyone has is collision: two agents editing the same file, clobbering each other, producing a tangle no one can unwind. It's a reasonable fear, and it's the one part of this that turns out to be cheap.

The fix is to give each agent its own private copy of the codebase — its own branch, its own working directory — so that nothing one agent writes can touch what another is reading. The underlying version-control tooling supports exactly this, several isolated copies of one repository sharing a single history, and it costs almost nothing to set up. Agents fan out into their own sandboxes, work without stepping on each other, and each produces a branch that becomes a pull request.

But isolation is not the same as independence, and this is where the trouble starts. Giving two agents separate rooms prevents them from bumping into each other. It does not make their tasks actually separable. You can put two people in soundproof booths and ask them both to write chapter three of the same book; they won't collide, and you'll still get two incompatible chapter threes. Isolation removes the symptom. It says nothing about whether the work was ever parallel to begin with.

## Decomposition is the real work

Whether two tasks can genuinely run in parallel is a property of the *work*, not the tooling, and it comes down to a simple test: the two tasks are independent only if they touch disjoint parts of the system and neither one needs the other's output. Change a shared type that lives on the boundary between two modules, and every task that reads that type is now downstream of yours. Touch the schema everything queries, and you've created an ordering constraint whether you declared one or not.

So the actual skill in running parallel agents isn't launching them. It's carving the objective into pieces whose edges don't overlap — and, just as importantly, recognizing the pieces that *can't* be carved apart and keeping them whole. A cross-cutting change is a single unit of work or it is a disaster; it is never something to split across two agents who will each solve half of it in mutually incompatible ways.

This carving is the part that doesn't parallelize. It's a planning act, done up front, by whoever understands the system well enough to see its seams. And it's unforgiving in a specific way: a bad decomposition doesn't fail quietly on one branch. It gets amplified. Hand a flawed breakdown to eight agents and you don't get one confused change to fix — you get eight, each internally plausible, that only reveal their disagreement when you try to merge them together. The leverage of parallelism cuts both ways. It multiplies good decomposition and it multiplies bad decomposition with exactly the same efficiency.

## The wall is review

Now suppose you've done the hard part. The work is cleanly carved, the agents are isolated, every task is genuinely independent. You launch ten. Ten agents produce ten pull requests. And every one of those pull requests has to be read, understood, and approved by a human before it can become part of the product.

That review is serial, and it runs at human speed. It does not care how many agents produced the queue. This is the wall, and it's a hard one: the throughput of the whole system is now governed not by how fast the agents can write code but by how fast a person can responsibly review it. Add more agents and you don't add throughput — you add depth to a queue that drains at a fixed rate, and you add risk, because while those pull requests wait their turn the trunk keeps moving and each unmerged branch drifts further from the thing it will eventually have to merge into.

> Picture a newsroom with fifty reporters and one editor. Hire fifty more reporters and the paper does not ship twice as much news. It ships exactly as much as before, because everything still funnels through the one desk that has to read each story before it runs — and now that desk is buried, stories go stale waiting, and the fresh ones contradict the ones still in the pile. The constraint was never the number of reporters.

This is Brooks's Law, the observation that adding people to a late software project makes it later, returning in a new costume. The mechanism is identical: beyond a small number, the cost of coordinating and integrating parallel work grows faster than the work the new units contribute. It was true when the units were programmers. It is true when the units are programs. Nothing about an agent repeals the arithmetic of integration.

## Cap it, and mean it

The practical consequence is almost rude in its simplicity: cap the number of concurrent agents to the number of changes a human can actually review, not the number that are technically parallel-safe. Those are very different numbers, and the gap between them is a trap. The plan might contain eight independent tasks; that does not mean you should have eight reviews open at once.

The cap that matters is on work *awaiting review*, not work in total. Let a small number of changes be in flight; as each one is reviewed and merged, a slot frees and the next queued task begins. Tasks wait their turn on the runway instead of piling up at the gate. And the right number is not fixed — it tracks where the bottleneck genuinely is. A batch of trivial, low-risk changes a reviewer can clear at a glance can run wider. Anything subtle, anything tightly coupled, anything where a mistake is expensive should narrow toward one-at-a-time, each change settling before the next begins.

## Put the human in exactly two places

If the constraint is human attention, the temptation is to spend it everywhere — to hover over each agent, approving its every step. That's the opposite of the answer. Hovering reintroduces a human into the fast path you were trying to speed up, and it stalls the whole machine on the slowest possible input. The discipline is to place human judgment where it changes the outcome and remove it everywhere else.

Two moments earn it. The first is approving the decomposition before any work starts — because a wrong breakdown is the expensive mistake, the one that gets multiplied, and it's far cheaper to catch on paper than across eight branches. The second is reviewing each finished change before it merges — the last line of defense, the thing you cannot delegate to the system that wrote the code. Between those two moments, the machinery should run without asking permission, because every mid-flight prompt is a stall, and a workflow that interrupts constantly is just a slow way to do one thing at a time. The art is making those two checkpoints strong enough that you can afford to trust the silence between them.

## The shape of it

Strip away the novelty and the pattern is familiar. Isolation is cheap and feels like the problem, so people solve it and think they're done. Decomposition is expensive and quiet, so people skip it and pay later. Review is fixed and human, so people flood it and slow down. None of this is new. It's the coordination cost of parallel work, the same cost that has always governed how many hands you can usefully put on one thing, showing up again now that some of the hands are software.

The machine was never the constraint. It's tempting to believe that faster, cheaper workers change the shape of the problem, but they only change its speed. What limits parallel agents is what has always limited parallel anything: how well the work can be divided, and how fast the results can be integrated back together. Get those two right and a swarm of agents is a genuine multiplier. Get them wrong and you've built, with remarkable efficiency, a very fast way to generate a backlog.
