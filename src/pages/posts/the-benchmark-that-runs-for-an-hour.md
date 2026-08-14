---
layout: ../../layouts/PostLayout.astro
title: "The Benchmark That Runs for an Hour"
description: "A thirty-second measurement captures a device at its best and never again. Everything that determines whether a feature can ship — throttling, duty cycle, the real cost of staying awake — only appears once the benchmark outlasts the thermal mass of the chip."
date: 2026-08-06
tags: ["edge-ai", "power", "thermal", "benchmarking"]
---

*Part 9 of a ten-part series on the engineering problems that define edge AI.*

Every other article in this series has quietly assumed something that deserves examining directly: that the numbers you measure describe the system you will ship. For latency and accuracy that assumption is mostly safe. For power and thermal behavior it is badly wrong, and the gap between a short benchmark and a device in someone's pocket is where a great many on-device features die.

## What thirty seconds cannot show you

Run a model in a loop for thirty seconds, record the average latency, and you have measured a device in a state it will almost never occupy: cool, idle, with every performance core available and no competing work.

Run the same loop for thirty minutes and the numbers move. Latency climbs, sometimes substantially. Not because anything in the software changed, but because the chip got hot and the system responded.

**Thermal throttling** is the mechanism, and it is worth understanding as deliberate design rather than failure. Silicon has a maximum safe operating temperature, and a phone has almost no room for a heatsink. What it has instead is thermal mass: the metal and glass absorb heat for a while, which is precisely why short bursts run at full speed. When that mass saturates, the operating system reduces clock frequencies to keep temperature in range. It will do this indefinitely, silently, with no error and no notification. The device gets slower and keeps working, because slower is the correct answer to hot.

The consequence for measurement is direct. A benchmark shorter than the device's thermal time constant measures the burst regime, which is the best the hardware will ever do. A feature that runs continuously lives in the throttled regime, which is where it will always be. These can differ by a factor of two or more, and there is no way to infer the second from the first.

The practical rule follows: **run the benchmark long enough for temperature to plateau, then measure.** Ten to thirty minutes is typical, and the shape of the curve during that window is itself informative — how fast it degrades, where it settles, and whether it settles at all.

## Energy is not power

A distinction that trips up nearly everyone at first, and that changes what optimizations look correct.

**Power** is the instantaneous rate of consumption. **Energy** is power integrated over time. Battery life depends on energy. Optimizing power alone can increase energy, and frequently does.

The illustration is straightforward. Imagine a task that can run on a slow, efficient core drawing one unit of power for ten seconds, or on a fast core drawing four units for two seconds. The first draws less power. The second consumes eight units of energy against the first's ten, and — more importantly — finishes sooner and lets the entire system return to a deep idle state that the first option kept it out of for an extra eight seconds.

This is **race to idle**, introduced in Part 4 and worth stating precisely here: finishing fast and sleeping deeply usually beats running slowly and staying awake, because idle states are dramatically cheaper than any active state, and reaching them sooner matters more than being gentle while active.

It also means intuitions about "lightweight" processing can be exactly inverted. A background task that wakes every second to do a tiny amount of work may consume far more energy than one that wakes once a minute and does sixty times as much — because the second spends fifty-nine seconds in a state the first never reaches.

## Duty cycle beats everything

Which leads to the single number that dominates battery life in an always-on system: the fraction of time anything is running at all.

The arithmetic is unforgiving in a useful way. A model running continuously at modest power will drain a battery faster than a model consuming ten times the power for one percent of the time. Nothing about the model explains this. The schedule explains it entirely.

The implication for effort allocation is the practical heart of this article, and it echoes Part 4 because it is the same truth from the measurement side. **When a system misses its power budget, the model is usually not the problem.** The higher-yield questions are: how often does this run, does it need to run that often, can the cheap gate reject more, can work be batched so the device wakes less frequently, can several subsystems be aligned to wake together rather than taking turns keeping the device alive?

Those questions routinely find an order of magnitude. Compressing the model further typically fights for a factor of two, and costs weeks.

## Measuring it honestly

Power measurement is harder than latency measurement, and the difficulty is worth acknowledging rather than working around.

**On-device estimates** — the counters and energy attributions the operating system provides — are convenient and approximate. They typically model consumption from activity counters rather than measuring current directly, and attribution between processes is imperfect. They are good for detecting large relative changes and unreliable for absolute figures.

**External measurement** — instrumenting the actual power rail — is accurate and requires hardware and effort. It is what you use when a number has to be defensible.

**Battery drain over hours** is the crudest method and the most honest, because it measures the thing users experience. Charge the device, run a realistic workload for several hours, measure the delta. It cannot tell you *which* component consumed the energy, and it cannot be argued with about how much was consumed.

A workable practice is to use on-device estimates for iteration and differential comparison, and to confirm the final answer with a long drain test under realistic conditions. What does not work is quoting on-device estimates from a short run as though they predicted battery life.

The methodology that makes any of these interpretable is comparative: **measure the device doing nothing first.** Baseline drain with your feature off, then with it on, under otherwise identical conditions — same screen state, same brightness, same network activity, same background applications. Absolute figures are nearly meaningless because so much else is consuming power. The delta is the thing you control and the thing you are responsible for.

## Cores are not interchangeable

Modern mobile processors contain cores of different designs — a few fast and power-hungry, several slower and far more efficient. The operating system decides where each thread runs, based on system-wide conditions you do not control.

This produces measurement noise that looks like randomness and is not. The same work, scheduled onto an efficiency core, may take several times longer than on a performance core. A benchmark run while the device is otherwise busy may land differently than one run on an idle device. Thermal pressure biases the scheduler toward efficiency cores, which is one reason latency degrades under sustained load even beyond the clock reduction.

For understanding a system, the useful response is to record which core type served each measurement where the platform exposes it, and to treat a bimodal latency distribution as a scheduling artifact rather than a mysterious property of the model. Two clean clusters in a latency histogram is almost always two core types, and reading it as one distribution with high variance will send you looking for a bug that does not exist.

## The measurements that predict shipping

Pulling this together, a small set of numbers tells you whether an always-on feature is viable — and none of them come from a short benchmark.

**Sustained latency after thermal plateau**, not burst latency. The number the feature will actually deliver in ordinary use.

**Energy per useful result** across a realistic day, as defined in Part 4 — total energy divided by correct detections, not energy per inference.

**Incremental battery drain over several hours**, measured against a baseline with the feature disabled.

**Duty cycle**: the fraction of time each tier of the system is active. Usually the most actionable number on the list, and usually the one nobody has.

**Skin temperature under sustained load**. A feature that makes a device uncomfortable to hold has failed regardless of what the power figures say, and thermal limits are frequently reached because of how a device *feels* rather than what the silicon can tolerate.

## The reframing

Latency benchmarking asks: how fast is this? It is a question with a stable answer, and a short measurement answers it adequately.

Power and thermal behavior ask a different question: **what does this cost to keep doing?** That question has no answer at thirty seconds, because at thirty seconds the device is still spending thermal mass it accumulated while idle. The honest answer only appears once the benchmark has outlasted that reserve and the device has settled into the state it will actually live in.

Which is the whole discipline, compressed: measure long enough that the device stops flattering you, and measure the delta against doing nothing at all.

---

*Next in this series: why "it runs on device" is a claim that has to be proven rather than a property you get for free.*
