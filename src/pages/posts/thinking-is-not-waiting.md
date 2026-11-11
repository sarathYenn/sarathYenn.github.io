---
layout: ../../layouts/PostLayout.astro
title: "Thinking Is Not Waiting: Running a CPU-Hungry Engine on a Shared Machine"
description: "A slow API call and a slow calculation look the same on a latency chart, but they need opposite fixes. A plain-language guide to cores, time limits, containers and pool sizes, and the quiet failure where answers get worse instead of slower."
date: 2026-11-11
tags: ["performance", "concurrency", "containers", "architecture"]
---

Many backends that call a large AI model also run a piece of software that does real calculation: a search engine for a game, a route planner, a scheduling solver, a fraud scorer that explores combinations. On a latency chart the two look alike. The model call takes seven seconds, the calculation takes one, and both show up as "slow steps". They are not the same kind of slow, and treating them the same way leads to some of the most confusing performance problems a team can hit.

This post explains the difference with plain examples, then walks through what it means when that calculating software shares a machine with other apps, runs inside a container, or is copied into a pool to serve more users.

## Two kinds of slow: the pizza and the sudoku

Think of a slow network call as **ordering a pizza**. Once the order is placed, you are not cooking. You are waiting. One person can wait for fifty pizzas at once, because waiting takes no effort.

Think of a slow calculation as **solving a sudoku**. Someone has to actually think the whole time. One person solves one sudoku at a time. Two sudokus take twice as long, and no amount of clever scheduling changes that.

In computer terms, the person is a **processor core**: one unit of the chip that can run one stream of instructions at a time. A typical laptop or small server has somewhere between four and sixteen of them.

This is the single most useful idea in the whole topic:

- **Async code, callbacks, threads and coroutines decide who waits.** They let a program wait for many pizzas without freezing.
- **They do not create more people who can think.** A calculation needs core time, and the only way to get more of it is more cores, or fewer other things competing for them.

A team that moves a heavy calculation into an async function and expects it to stop slowing the server down has changed who is waiting, but not who is thinking.

## Software that gets better the longer it runs

Some calculations have a natural end: sort this list, resize this image. Others never really finish. They keep improving their answer for as long as you let them run. Computer science calls these **anytime algorithms**.

Search engines for games are the textbook case. Imagine software deciding the best next move. It looks one move ahead, then two (every move times every reply), then three, and so on. Each extra step multiplies the work, so it simply keeps digging deeper until someone tells it to stop. Route planners, optimisation solvers and many search-based AI systems behave the same way.

So you have to give it a stopping rule, and there are only two kinds:

| Stop when... | You always get | When the machine is busy |
|---|---|---|
| **A time limit** is reached (say, one second) | An answer on time | The answer is **weaker**: it got less thinking done |
| **A depth or quality target** is reached | The same quality | The answer is **slower**: it takes longer to get there |

Neither is wrong. But the choice decides how your system fails, and the first one fails silently.

## The quiet failure: worse, not slower

Here is the trap. Most teams choose a time limit, because users want an answer on time. Then the machine gets busy. Maybe other apps run on the same server, maybe more users arrive, maybe a nightly job kicks in.

The calculating software still answers in exactly one second. The latency dashboard stays green. Nothing errors. But in that one second it got only a third of a core instead of a whole one, so it looked a third as far ahead. In a simple experiment on an eight-core laptop, one search running alone examined about five times as many positions in its second as each of sixteen searches running at the same time. Same deadline, a fifth of the thinking.

Often the answer comes out the same anyway. In easy cases a shallow look finds the same best option. In hard cases, the ones where the user most needs a good answer, it can miss what matters.

The fix is to **measure the work, not just the time.** Most search software can report how deep it went or how many options it examined. Log that number next to the latency, and alert when it drops. A sudden drop in "positions examined per second" is the signal that the machine is too busy, long before anyone notices worse answers.

## A core is shared, not owned

It is natural to say "the engine needs a full core", and it is easy to hear that as "the engine gets a core of its own". It does not. On a normal operating system, nothing owns a core. The system shares all the cores among every program that wants one, many times a second, like a teacher handing a single marker around a room.

A program that runs one stream of work can use **at most one core's worth of time per second**. That is its ceiling, not a reservation. What it actually gets depends on who else is asking:

- On a quiet machine it gets nearly the full second.
- Next to a video encoder, a build server or another heavy app, it might get half.
- On a very busy machine, much less.

The number of cores matters because it tells you **how many heavy programs can run at full speed at the same time** before they start slowing each other down. Eight cores means roughly eight. The ninth makes everyone share.

Two practical levers follow:

- **Priority.** Operating systems let you mark a program as less important (the long-standing Unix tool for this is called `nice`). When both want the processor, the less important one yields. That's useful when the calculation shares a machine with background work that can wait.
- **Not all cores are equal.** Many modern chips, in laptops and especially in phones, mix fast "performance" cores with slower, power-saving "efficiency" cores. Two programs that each "get a core" may get very different amounts of thinking done, depending on which kind they land on.

## Containers: the box has its own walls

Containers add a layer that surprises many teams, in two ways.

**First, on a laptop the container usually runs inside a small virtual machine,** a computer simulated by software, with its own fixed number of processors and memory set in the container tool's settings. Imagine a team that runs several services in containers on a desktop machine with eight cores, while the container tool's virtual machine is set to two. Every containerised service, including the heavy calculation, now shares two processors, however idle the other six are.

**Second, the container may not tell the program the truth about its limits.** You can give a container a CPU limit in two ways:

- **A share of time** ("this container may use up to two cores' worth of time"). The program can run on any processor, but it is held back once it has used its allowance.
- **Specific processors** ("this container runs only on processors 0 and 1"). This is called pinning.

Many programs size themselves by asking "how many processors does this machine have?" Inside a container with a time-share limit, that question often returns the number of processors the whole machine or virtual machine has, not the allowance. A program that sees sixteen processors and starts sixteen workers, inside a box allowed two, has built the sixteen-searches-at-once slowdown into itself. Java's runtime famously made this mistake for years, sizing thread pools by the host's processors, until its later versions learned to read the container's limits. Several other runtimes still only count the processors they are pinned to, and ignore time-share limits entirely.

The safe habit is simple: **set the number of workers explicitly in configuration, to match the limit you gave the container.** Don't let the program guess.

## Sizing a pool: match the cores, not the users

When one copy of the calculating software is not enough, the natural next step is a pool: several copies, each serving one request at a time. The question is how many.

| Pool size | What happens |
|---|---|
| **One copy** | Every answer gets full quality, but requests queue. Under load, people wait. |
| **About one copy per fast core you are allowed** | Requests run side by side at near-full quality. The sweet spot. |
| **One copy per user** | No queue, but copies outnumber cores. Every answer quietly gets weaker, and the dashboard shows nothing wrong. |

The pool should follow the **processors you have**, not the **people you serve**. When users outnumber processors, a short queue with full-quality answers beats no queue with degraded ones. Pair the pool with a bounded wait: if a request cannot get a copy within a few seconds, tell the user plainly rather than letting the queue grow forever.

Notice what the pool does **not** need to cover: the slow network calls. A request that spends seven seconds waiting for an AI model holds no processor during that time. A server can have dozens of those in flight on two cores. Only the thinking part needs the careful count.

## The same idea on a phone

Edge devices make all of this sharper. Imagine an anti-scam assistant that scores incoming messages on the phone, or an identity app that checks a face for liveness before upload. The on-device model is a calculation, not a wait. It competes with the camera, the screen and every other app. The phone may move it to an efficiency core to save battery, or slow every core down when it gets warm.

The same rules apply. Keep the calculation off the thread that draws the screen. Prefer a stopping rule that fails the way you want (a time limit for a live camera feed, a quality target for a security decision). Measure how much work actually got done, not just how long it took. And never assume the program gets a whole fast core just because one exists.

## A checklist

1. **Label each slow step: waiting or thinking?** Async helps waiting. Only cores help thinking.
2. **For software that improves with time, choose the stopping rule on purpose.** A time limit means busy equals weaker; a quality target means busy equals slower.
3. **Log the work done** (depth, options examined, iterations) next to latency, and alert on drops.
4. **Treat cores as shared.** Know what else runs on the machine, and use priority when the neighbours can wait.
5. **Know your container's real limit**, including the virtual machine underneath it on a laptop.
6. **Set worker and pool counts in configuration**, matched to that limit. Never trust a program's own guess inside a container.
7. **Size pools by processors, not users**, with a bounded wait and an honest message when it is full.
8. **On devices, expect slower cores and heat throttling**, and pick the failure mode that suits the feature.

## Further reading

- [Anytime algorithm](https://en.wikipedia.org/wiki/Anytime_algorithm) — the general idea of computations that improve the longer they run.
- [Iterative deepening](https://www.chessprogramming.org/Iterative_Deepening) — how game search engines dig one level deeper at a time, and why they can stop at any moment.
- [Docker: resource constraints](https://docs.docker.com/engine/containers/resource_constraints/) — time-share limits versus pinning to specific processors.
- [Linux CFS bandwidth control](https://docs.kernel.org/scheduler/sched-bwc.html) — how the kernel enforces "this many cores' worth of time" under the hood.
- [Python's `os` module](https://docs.python.org/3/library/os.html) — compare its two processor-count functions to see the "which count?" problem in one runtime.
- [Brendan Gregg: the USE method](https://www.brendangregg.com/usemethod.html) — a simple way to check utilisation, saturation and errors for every resource, cores included.
- [`nice` manual page](https://man7.org/linux/man-pages/man1/nice.1.html) — the classic way to tell the system which program matters less.
- [Kotlin: coroutine context and dispatchers](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html) — the separate pools for waiting work and thinking work, in one language.
- [Apple: Dispatch quality of service](https://developer.apple.com/documentation/dispatch/dispatchqos) — how a phone decides which work gets fast cores.
- [Android: power and performance](https://developer.android.com/topic/performance/power) — why heavy work on a phone gets slowed or moved.
