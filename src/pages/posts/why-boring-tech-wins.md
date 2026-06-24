---
layout: ../../layouts/PostLayout.astro
title: "Why Boring Technology Usually Wins"
description: "Every shiny new tool has a cost that doesn't show up on the benchmark page. Here's how I think about technology choices at scale."
date: 2026-06-23
tags: ["architecture", "engineering-decisions"]
---

Every six months, something new promises to solve the problem your current stack "isn't designed for."

I've shipped systems on Kafka, Kinesis, Pulsar, and a few things that no longer exist. The pattern I keep seeing: teams that pick boring, well-understood technology ship faster, debug faster, and sleep better.

## What "boring" actually means

Boring doesn't mean old. It means:

- **Deep documentation** — someone has already hit your edge case and written about it
- **Predictable failure modes** — you know what breaks and roughly when
- **Ecosystem depth** — the libraries, the tooling, the StackOverflow answers all exist

Redis isn't boring because it's old. It's boring because after 15 years, there are very few surprises left in it.

## The hidden cost of novelty

When you adopt a new tool, you're not just adopting the technology — you're adopting:

- The cognitive load of learning it under pressure
- The operational burden of running something your team doesn't know
- The on-call risk of debugging something at 3am with no runbook

Benchmarks don't show you what happens when the thing breaks in production at 2am and the only person who knows how it works is on PTO.

## When to actually pick something new

That said, the answer isn't "never change anything." The question I ask:

> Does this new tool solve a problem we've already demonstrated we have, at a scale we've actually reached?

If the answer is yes and the team has time to build operational fluency before it matters, go for it. Otherwise, reach for the tool that's already working somewhere else in your stack.

---

The best technology decision I ever made was talking my team out of a distributed saga framework in favor of a single Postgres table with a `status` column. We shipped in a week. It's still running.
