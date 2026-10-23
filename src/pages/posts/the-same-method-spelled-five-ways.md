---
layout: ../../layouts/PostLayout.astro
title: "The Same Method, Spelled Five Ways"
description: "Build a graph that maps a code change to what it can break, and the hard part isn't walking the graph — it's deciding what a call actually points to. A method has one definition and many spellings, and getting that identity wrong quietly wrecks precision."
date: 2026-10-22
tags: ["static-analysis", "developer-tools", "compilers"]
---

Start with what the thing *is*, because the whole post hinges on it.

Imagine a graph where each node is a piece of code — a function, a method, a field — and an edge means "if this changes, that could change too." Build that graph over a whole codebase and you can answer a genuinely useful question: given a diff, what could it break? You find the nodes the diff touched, walk the edges outward, and arrive at the set of things downstream — ultimately, the user-facing surfaces a tester should exercise. Tools in this family go by different names — affected-targets analysis, change-impact analysis — but the shape is the same, and so is the thing that makes or breaks them: **precision.** Draw too many edges and every small change lights up the entire application, so the output is noise and nobody trusts it. Draw too few and you miss the edge that mattered, and something ships broken.

Most of the edges come from a relationship that sounds trivial: *this piece of code uses that symbol.* Function A calls function B; A depends on B; draw the edge. The trouble is hidden inside the word "uses." When the source says `a.b.c()`, which definition, out of everything in the codebase, does `c` actually refer to? Answer that wrong and you've drawn an edge to the wrong place — a false dependency that will light up the wrong surface on some future change. Nearly all the precision in a tool like this lives not in the graph walk but in that one unglamorous act: resolving a name to the thing it names.

Here are the three ways that resolution goes wrong, and the single idea that fixes all three.

## A method has one definition and many spellings

A method is defined once. But the callers who use it write that usage in wildly different ways. One calls it directly. One pulls the containing object into a local variable first and calls through that. One destructures it out. One imports it under a renamed alias. One reaches it through a chain of nested accessors. Five call sites, five different surface spellings, all pointing at the same single definition.

If your resolver keys identity on *how the call is written*, those five callers fracture. Some resolve to the right node, some resolve to nothing, some resolve to the wrong node that happens to share a spelling. Now a change to that one method reaches only the subset of callers who happened to spell it the way your resolver expected. The others silently miss the edge. Your recall quietly has a hole in it, shaped exactly like the spellings you didn't anticipate.

The invariant you actually want is easy to say and clarifying to hold onto: **a method means the same node to every caller, regardless of how that caller spelled the access.** Identity is anchored on the receiver — the thing being called — not on the caller's syntax. Every spelling that ultimately reaches the same definition must resolve to the same node. Once you state it that way, "handle aliases" and "handle nested access" stop being a pile of special cases and become one requirement: normalize every spelling down to the receiver's identity.

## A member access is not a top-level function

This is the subtle one, and it moves precision in the dangerous direction — it adds *false* edges rather than dropping true ones.

Picture two unrelated things that happen to share a short name. One is a method on some object — you reach it as `widget.refresh()`. The other is a standalone, top-level function named `refresh`, defined in a completely different corner of the codebase, with nothing to do with the widget. If your resolver indexes everything by its bare last-segment name and looks `refresh` up in that flat table, `widget.refresh()` can bind to the top-level `refresh`. Now a change to that unrelated standalone function draws an edge to the widget, and every surface the widget feeds lights up on a change that could not possibly affect it.

The rule that prevents this is almost embarrassing to state once you see it: **a member access resolves only to a member definition.** A `.refresh` is never the standalone `refresh`. They live in different namespaces — one is a property of something, the other is a free name — and conflating them because they share letters is a category error. It's embarrassing precisely because it's obvious in hindsight, and easy to walk straight into when you've built a single convenient lookup table keyed on the last segment of every name. The convenience is the trap.

## A reference to a type is not a reference to all its behavior

The third trap is about aggregation. Consider a class with many methods, used all over the codebase — often merely as a *type*. A variable is declared to be of that class; a parameter is annotated with it. That's a reference to the class, yes. But to *what about* the class?

The lazy answer is: to all of it. Draw an edge from the class to everyone who mentions it, and now a change to any single method of that class lights up every place that so much as typed a variable as that class — including places that never call that method, that only ever touch one unrelated corner of it. One popular class becomes a hub that fans every internal change out across the entire application. Precision collapses.

> A library card identifies you to the library. It does not mean you have read, or are responsible for, every book on the shelves. Naming the building is not the same as using what's inside it.

The fix is a principle that generalizes well beyond classes: **resolve to the finest thing that can change independently.** A bare reference to the class-as-a-type binds only to the class's shell — its constructor, its shared declaration — not to the union of all its methods. A specific method is reached only by a caller who actually calls *that* method. Then a change to one method reaches exactly the callers of that method, and the class stops being a hub. The same idea applies one level down, too: a change inside a method resolves to that method, not to the whole class that encloses it; a change to one field of a configuration object resolves to that field, not to every reader of the object. Impact should flow at the grain of what can actually vary on its own.

## The fix that worked and hurt at the same time

A confession, because it's the most useful part.

Faced with the collision problem — the member access binding to the wrong same-named thing — there's an obvious-seeming fix: disambiguate by matching a *longer* piece of the name. Don't just match the last segment; match the chain, so a qualified access binds to the definition whose fuller path lines up. It's a natural instinct and it is, in a narrow sense, a cleverer string match.

It worked. It recovered the cases that had been missing. It also, in the same change, re-introduced a batch of false edges that an earlier fix had eliminated — because capturing the longer chain *also* emitted the shorter parent segment along the way, and that parent re-merged a set of things we had deliberately split apart. The net effect was ambiguous: a recall gain, a precision loss, and — this is the important part — a mechanism I could not fully explain in the moment. Why exactly did the parent re-merge? Which cases, and how many?

That inability to explain is the signal. A change that both helps and hurts by a mechanism you can't articulate is not a tuning dial to be nudged; it's a stop sign. I reverted it. The correct fix turned out to be narrower and more principled than a smarter string match — resolve member accesses only against member definitions, and anchor identity on the receiver — and it recovered the missing cases *without* reopening the old wound, because it was addressing the actual shape of the problem rather than pattern-matching around its symptom. The lesson I keep relearning: when a fix is a clever string manipulation, be suspicious. The real fix is usually a clearer idea, not a longer regular expression.

## The shape of it

Three traps, one underlying subject: identity. What does this name point to?

- **Anchor identity on the receiver, not the spelling.** Every way a caller can write a reference to the same definition must resolve to the same node, or your recall develops holes shaped like the syntaxes you forgot.
- **Keep namespaces separate.** A member access is not a same-named free function. Resolving both out of one flat table by last segment is how unrelated code ends up wired together.
- **Resolve to the finest independently-changeable thing.** A type reference is the shell, not every method; a method change is the method, not its class; a field change is the field, not its object. Coarse identity turns popular code into hubs that fan every change everywhere.
- **Distrust the clever string fix.** If a resolution change both helps and hurts by a mechanism you can't explain, revert it and find the idea underneath. The durable fix is almost always a sharper concept, not a longer match.

---

What surprised me most, building one of these, was where the precision actually came from. I expected it to live in the analysis — in how cleverly the graph was walked, how impact was propagated. It didn't. The walk was the easy part. Almost every real gain came from the quiet layer underneath it, the part that decides what a name means: that a method is one thing to all its callers, that a dotted access is not a bare function, that naming a type is not invoking its every behavior. Get identity right and the graph more or less walks itself. Get it wrong and no amount of clever propagation will save you — you'll just be faithfully spreading impact along edges that were never really there.
