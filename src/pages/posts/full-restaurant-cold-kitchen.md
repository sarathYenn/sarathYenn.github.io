---
layout: ../../layouts/PostLayout.astro
title: "A Full Restaurant With a Cold Kitchen"
description: "A background system can report itself at capacity while the expensive resource it exists to protect sits idle. The usual cause is the same one twice over: the limit counts the wrong thing, and work that is only waiting is allowed to hold a seat."
date: 2026-07-22
tags: ["concurrency", "architecture", "scaling", "backend"]
---

Picture a restaurant with twelve tables and a single rule at the door: never seat a thirteenth party. It is a disciplined rule, trivial to enforce, and on a busy night the maître d' turns people away into the drizzle with a clear conscience. The house is full. Twelve tables, twelve parties, not one more.

Now walk back to the kitchen and find it cold. Half the burners unlit, the chef standing at an empty pass with a spoon and nothing to do. A full restaurant, asking almost nothing of the one room that decides how much food it can actually make.

This is not really a story about restaurants. It is the most common shape of failure I have come to recognize in systems that do work out of sight — the kind where a request arrives, gets handed to a pool of workers, and some slower process grinds away in the background. Such systems have a habit of reporting themselves at capacity while the very thing they exist to protect sits idle. Once you have seen the restaurant, you start seeing it everywhere, and you start noticing that the same two mistakes are almost always underneath it.

## Occupied is not busy

Go back and watch a single table. A man sits alone. His menu is closed, his water glass sweats, and he is waiting — for a companion whose carriage is stuck across town — and he has decided, courteously, to order nothing until she arrives. He wants nothing from the waiter, who drifts past him a dozen times with nothing to carry. He wants nothing from the kitchen, which cooks him nothing at all. He simply occupies table nine, for an hour, the way a stone holds a place in a river.

He is occupied. He is not, by any measure that produces dinner, busy.

Systems are full of this diner. A unit of work that has started but is now waiting — on a person to decide something, on a slow reply from another service, on a lock, on anything — looks identical, from the outside, to a unit of work that is grinding hard. Both hold a slot. Both count against the limit. Only one of them is doing anything at all.

The confusion runs deep enough that the very word we reach for hides it. "Blocked" can mean three different things, and they are not the same. It can mean a worker frozen solid, unable to do anything else while it waits — the crude, expensive kind. It can mean the scheduler's attention is tied up, so nothing else gets a turn. Or it can mean only that the worker's *seat* is spoken for, while its attention and the machinery behind it are entirely free. Modern asynchronous runtimes attacked the first kind loudly and the second kind quietly, and left the third kind almost completely alone. A task merely awaiting a slow answer no longer freezes anything; the runtime cheerfully serves a thousand other things while it waits. But the task is still alive. It still holds its place in line. And most concurrency limits count places in line.

So you arrive at the paradox: a system doing nothing, at capacity. Every worker is "busy" in the only sense the limiter can perceive — occupied — and idle in the only sense that produces output. The queue outside keeps growing in the rain.

## Cap the fire, not the chairs

The maître d' built his whole discipline around tables. But a table is the *cheap* resource. It is wood and linen; you could set another against the far wall in an afternoon. The scarce, irreplaceable thing — the true ceiling of the house — is the kitchen: the burners, the oven, the single pair of hands at the pass. And he never counted the kitchen at all. He counted chairs, because chairs are the thing you can watch fill up.

This is the diagnosis under most "full but idle" systems. The limit has been drawn around the resource that is easy to observe rather than the one that is actually scarce. A cap on the number of concurrent workers *feels* like a cap on load, and it is one of the most natural knobs to reach for, because occupied workers are right there in the dashboard, filling up. But if a worker spends most of its life waiting, the count of workers tells you almost nothing about how hard the expensive machinery underneath is running. You have capped the seats and left the fire entirely ungoverned — which means that on the night the fire genuinely can't keep up, the limit you were relying on was never watching the thing that failed.

The remedy has two halves, and both cut against the instinct. The cheap resource — the seats, the worker slots — should be *generous*. Add chairs. They cost almost nothing, and being stingy with them is precisely what leaves paying guests in the rain while the kitchen idles. The expensive resource — the kitchen, the external call that costs real money or trips a real rate limit — is where the tight, deliberate cap belongs, drawn directly around the machinery you actually cannot buy more of on short notice. Two limits, on two resources, sized in opposite directions. Teams that conflate them into one number nearly always end up throttling the wrong one, and they discover which one only when the bill or the outage tells them.

## Don't hold the table to think

There is a question the whole scene has been quietly begging. Why was the man holding a table at all? He was not eating. He was deciding, and waiting. The table did nothing for him but exist beneath his elbows while he watched the door.

The change is almost embarrassing to state and genuinely hard to build: a guest who is only deciding does not hold a table. Take his coat and his intention — the wine he will want, the dish for when she arrives — write it on a slip, and give him the run of the bar. The table goes back into the world. The instant he is ready to truly eat, the instant he needs the *kitchen*, seat him again, slip in hand, exactly where the story left off.

In a system, that slip is the entire trick, and the reason work-that-waits so often gets written as work-that-holds-a-slot is that holding the slot is the lazy way to keep the state. A half-finished job's progress lives in the worker's memory — its variables, its position in the flow — and as long as the worker sits there, all of it is simply *present* when the wait finally ends. Freeing the seat means surrendering that convenience. You have to write the state down somewhere durable, hand the worker back, and be able to reconstruct precisely where you were when the signal to continue eventually arrives.

That reconstruction is the real cost, and it is not free: a flow written as one long function that pauses at each step becomes a small state machine driven by short, self-contained bursts of work, with the in-progress state living outside any single worker. It is more moving parts, and it demands more care — every resumption has to be idempotent, because a slip can be handed back twice. But it buys the one thing nothing else will. The seat is held only while the kitchen is actually cooking, and never for a second while a human deliberates. Any workflow with a person in the loop eventually meets this wall, and the ones that get past it are, without exception, the ones that stopped keeping progress inside a worker that was doing nothing but wait.

## What the maître d' learned

He kept his discipline to the end of his days; he simply learned, one slow Tuesday, what he had been guarding all along. The house was never limited by its chairs — chairs are cheap, and you add them freely. It was limited by the fire, and the fire was the thing he had never thought to count. Occupied was never the same as busy. And no one, guest or worker, should ever hold a table only to think.
