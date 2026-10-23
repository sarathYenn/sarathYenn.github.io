---
layout: ../../layouts/PostLayout.astro
title: "When the Benchmark Is the Bug"
description: "A change-impact tool's precision score sat stuck for weeks, and every instinct said the analyzer was wrong. The analyzer was fine. The ruler was broken — in three different ways, each of which looked exactly like a product bug."
date: 2026-10-22
tags: ["static-analysis", "testing", "engineering-decisions"]
---

Consider a tool whose whole job is to answer one question: given a change to a codebase, which user-facing surfaces could it break? A developer edits some code, and the tool returns a short list — these three screens, that one report — so a tester knows exactly what to exercise. The value of such a tool is entirely in the quality of that list. Too long and nobody reads it; too short and something ships broken. So you measure it. You assemble a corpus of past changes whose true impact you already know, you run the tool over each one, and you compute precision and recall.

Then the number comes back mediocre, and it won't move. You sharpen the analysis. You add a smarter resolver. You chase down edge cases. The number twitches by a point and settles back. Weeks pass. The analyzer is not the problem. The benchmark is — and the reason it took weeks to see is that a broken benchmark produces a number that looks, in every respect, like a report card on your product. It is actually a report card on your ruler.

There are three distinct ways the ruler goes wrong. Each one is worth naming, because each one masquerades as a different product defect.

## Trap one: the coordinates don't line up

The most seductive version. Your corpus is built from historical changes — real edits made months ago. Your tool, meanwhile, analyzes the *current* state of the code. So when the harness takes an old change and asks "which part of today's code did this touch?", it answers using coordinates that have since moved.

A change that once edited a function near the top of a file now points into the middle of a different function, because the file grew by two hundred lines in the interim. The harness faithfully feeds your tool the old line numbers; your tool faithfully resolves them against the new layout; the result is attributed to the wrong piece of code. Every such case shows up in your metrics as the tool being imprecise. It is not. It is being fed a map of a city that has been rebuilt.

This is the one that cost us the most, because the fix for the *apparent* bug — "make the resolver more precise" — is real, useful work that does nothing for the number, which keeps you believing you're on the right track.

The way out is to stop measuring against drift. Build a second corpus where each case is expressed against *the same snapshot the tool actually indexed* — so the coordinates are guaranteed to line up. Then compare the two. And here is the part worth internalizing: run the comparison even when you're sure you know the answer. We expected the aligned corpus to show a big jump. It showed no change at all. The two scored identically.

That null result was the most valuable measurement in the whole effort. It falsified the comfortable hypothesis — "the number is low because of drift" — in a single experiment. If we had simply switched to the aligned corpus and watched the number stay flat, we might have shrugged. Because we ran it as an explicit A/B, the flat line *meant* something: drift was not the cause, and a month of planned drift-fixing would have been a month spent polishing a non-problem. The discipline is to design the experiment that could prove your favorite explanation wrong, and to be genuinely pleased when it does.

## Trap two: the ruler is made of the thing it measures

This one is quieter and more dangerous, because it makes the number go *up*.

Every time the tool produced an answer that wasn't in the expected set, there was a nagging pull to do the easy thing: add that answer to the expected set. After all, maybe the tool was right and the corpus was incomplete. Sometimes that's even true. But as a reflex it is poison, because it is the benchmark equivalent of painting the bullseye around wherever the arrow landed. Do it consistently and your precision climbs toward a hundred percent while measuring nothing but its own output.

> A watch is only useful because it is set against something outside itself — a signal from an atomic clock, the rotation of the earth. A watch you "correct" by declaring that whatever time it currently shows is the right time is not a watch. It is a decoration that always reads noon.

The rule that keeps the ruler honest is simple to state and hard to hold: **ground truth must be derived independently of the system under test.** When the tool and the expected answer disagree, you do not edit the expected answer to match. You go read the actual source code and work out, by a completely separate path, what the right answer *is*. Only then do you decide who was wrong.

What's striking is how often, under that discipline, the expected answer turns out to be the thing that's wrong. A surface you'd marked as "not affected" turns out to render a component the change touched; a surface you'd required turns out to pull its data through a path that the change never reaches. The tool was right; your hand-authored truth was sloppy. You *cannot* discover that if your correction procedure is "make the truth agree with the tool." The whole point of an independent oracle is that it can tell you the tool was right — and that information is lost the moment the oracle is derived from the tool.

So the correct answer going up is not automatically good news. It's only good news if the thing it's being graded against was built without looking at it.

## Trap three: the flat line you don't want to report

The third trap is not about the harness at all. It's about what you do with a result you didn't like.

At one point we were certain a particular refinement — resolving impact at a finer grain than whole files — would lift precision meaningfully. We built it. The number didn't move. Not because the refinement was broken, but because of a structural fact about that particular codebase: its files already mapped almost one-to-one onto user-facing surfaces, so resolving *within* a file landed on the same surface the coarser method already did. The finer grain was correct, general, and — here — inert.

The temptation is to quietly drop that from the write-up. A flat line is a boring slide. But "this didn't help, and here is the precise structural reason it didn't" is a result, and often a more useful one than a win, because it tells you *where* the refinement would pay off: on a codebase whose files each serve many surfaces, exactly the case this one wasn't. Burying the null turns a finding into a gap. Reporting it turns it into a map of when the technique is worth reaching for.

## The checklist

If you're building anything whose output is a measured prediction — an impact list, a classification, a ranking — the distilled discipline is short:

- **Line up the coordinates.** If your corpus is historical and your tool analyzes the present, you are measuring drift as if it were error. Build a corpus expressed against the exact state the tool indexed, and measure there.
- **Run the experiment that could prove you wrong.** Don't just switch to the better harness and glance at the number. Set it up as an A/B against the old one, so a flat line carries information instead of a shrug.
- **Derive truth independently.** When the system and the expected answer disagree, investigate the source, never patch the expectation to match. A metric graded against the system's own output climbs to perfection and means nothing.
- **Expect the truth to be wrong sometimes.** An independent oracle earns its keep precisely when it reveals the tool was right and your hand-written answer was lazy. That signal only exists if the oracle never looked at the tool.
- **Publish the null.** A refinement that didn't move the number, with the structural reason why, tells you where it *would* — which is worth more than hiding it.

---

None of this is about the analyzer. That's the whole point. For weeks the mediocre number felt like a verdict on the thing we were building, and every honest attempt to improve the thing left the verdict unchanged. The number was real; it just wasn't about what we thought. It was reporting, in order, that our coordinates were stale, that our answer key was half-authored, and that our expectations about where precision would come from were wrong.

A measured system has two artifacts, not one: the system, and the instrument you judge it by. The instrument needs calibration as much as the system needs code, and it fails in ways that are harder to see, because a broken ruler doesn't throw an error — it just quietly hands you a number with your product's name on it. The hardest part of building something you can trust is not the building. It's learning to distrust the measurement until you've tried, in earnest, to break the ruler first.
