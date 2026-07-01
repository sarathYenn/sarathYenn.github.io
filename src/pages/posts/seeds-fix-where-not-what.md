---
layout: ../../layouts/PostLayout.astro
title: "Seeds Fix Where, Not What"
description: "A fixed seed makes image generation reproducible — but not editable. Here's why keeping the seed and tweaking the prompt works for some changes and scrambles the whole image for others."
date: 2026-07-01
tags: ["ai", "diffusion-models", "image-generation"]
---

There's a tempting workflow in image generation: you land on a picture you almost like, so you keep the seed fixed, change two words in the prompt, and generate again — expecting the same image with just the one thing you asked for, changed.

Sometimes it works exactly like that. Sometimes the entire image reorganizes and you're left staring at a completely different composition, wondering what a seed is even for. The gap between those two outcomes isn't luck. It's a direct consequence of how these models turn noise into pictures, and once you can see the mechanism, you can predict which edits will hold and which will blow up before you spend the generation.

## The version-lock misconception

The seed makes generation reproducible: same seed, same prompt, same settings, and you get the same image back, bit for bit. From there it's an easy leap to assume the seed is a kind of version lock — that it pins the *image*, so a small change to the prompt should produce a small change to the image.

That leap is the mistake. Reproducibility and editability are different properties, and the seed only gives you the first one. Understanding why is the whole point.

## What a seed actually fixes

A generation doesn't start from a blank canvas. It starts from a field of pure random noise, and the seed is simply what determines that noise. Same seed, same starting noise.

The important part is what that noise *is*: a spatial prior. It biases roughly **where** things will end up — the broad layout, the placement of masses, the skeleton of a pose, where the horizon sits. It says almost nothing about **what** those things are. The seed is a lattice of tendencies about arrangement, not a description of content.

So when you fix the seed, you're fixing the starting point of a journey, not its destination. The prompt is what walks the image from that starting noise to a finished picture — and two different prompts can walk the same starting point to wildly different places.

## Generation is a trajectory, and trajectories can diverge

Think of the denoising process as a path. At each step the model looks at the current half-formed image and nudges it a little closer to something coherent, and the direction of every nudge is steered by the prompt. Run all the steps and the path terminates at a finished image.

Now change the prompt slightly. You've changed the steering — not by much, but at every step. From the same starting noise, the new path begins right next to the old one. Two things can happen from there:

- The paths stay close the whole way down, and you get the surgical edit you wanted: same picture, one attribute changed.
- The paths start close but pass near a fork, and the tiny difference in steering is enough to send them down different branches. Downstream, they end up nowhere near each other.

This is sensitivity to initial conditions, with a twist. Usually we think of that sensitivity in terms of the starting point — and here the starting point is pinned by the seed. The thing being perturbed is the *prompt*. A small prompt change is a small perturbation to the entire steering field, and near a fork, small perturbations have large consequences.

> Imagine two hikers starting from the same trailhead, given slightly different compass bearings. On open ground they end up a few feet apart. But if a ridge runs between them, that tiny difference in bearing puts them in different valleys.

The seed picks the trailhead. The prompt picks the bearing. Whether your edit stays local depends entirely on whether there's a ridge nearby.

## Coarse-to-fine is the whole story

Here's the part that turns this from an interesting metaphor into something predictable. The denoising path isn't uniform — it does different work at different stages.

Early steps operate on an image that is still mostly noise. There's very little committed detail, and the model is making the biggest, coarsest decisions: overall composition, how many major masses there are, where they sit, the global structure. At this stage there's so little signal in the image that the **seed dominates** — the starting noise is most of what there is to work with.

Late steps operate on an image whose structure is already settled. The model is no longer deciding where things go; it's deciding what they look like — texture, color, lighting, surface finish, style. Here the **prompt dominates**, because the structure is locked and only the character of it is still in play.

That split predicts everything:

- An edit that targets a **late-stage, high-frequency** attribute — a warmer palette, softer lighting, a different rendering style, a change of material — lands after the seed-set structure is already committed. The composition survives untouched and only the surface changes. **This is where same-seed editing shines.**
- An edit that targets an **early-stage, low-frequency** attribute — adding or removing a subject, changing a pose, changing how many things are in frame, relaying out the scene — collides with the structure the seed already committed to in the first few steps. Now the seed and the new prompt are fighting over the same early decisions. Either the edit loses and gets ignored, or it wins and drags the whole composition into a new arrangement.

**The rule that falls out of this:** keeping the seed and tweaking the prompt works precisely when your tweak changes the *surface* of the image, not its *structure*. "Same scene, warmer light" is a surface edit and it will hold. "Same scene, but now there are two people" is a structural edit and the seed can't protect it.

## The seed is a weak anchor; attention is the strong one

If structure is what you want to preserve, it's worth asking what actually holds structure in place — because it turns out the seed is a surprisingly weak grip on it.

When you regenerate with a new prompt, the model doesn't consult the old image. It runs a fresh pass and recomputes, from scratch, which regions of the canvas pay attention to which parts of the prompt. That internal map — the correspondence between image regions and prompt concepts — is what really determines where things land. A fixed seed gives the fresh pass the same starting noise, but it does nothing to preserve that correspondence. So the structure is free to move, even with the seed pinned.

This is why the more surgical editing approaches don't rely on the seed at all. They work by **carrying over the internal attention pattern** from the original generation and letting only the edited part of the prompt change what fills it. Freeze *where each concept attends*, swap the concept, and the layout stays put while the content changes. Fixing the seed is a necessary starting point, but the attention pattern is the load-bearing anchor. Reuse the seed and you've reproduced the trailhead; reuse the attention and you've reproduced the map.

## Some seeds are fragile

There's a wrinkle that surprises people: whether an edit holds isn't purely a property of the edit. It's partly a property of the specific seed.

Some seeds land you in a stable composition — one that's sitting comfortably in the middle of a basin, well away from any fork. Nudge the prompt and the image barely flinches. Other seeds land you on a knife-edge, a composition that's only marginally the winner over some very different arrangement. It looks fine, but it's balanced on a boundary, and *any* prompt change — even one that should be trivially local — is enough to tip it over into the other arrangement entirely.

So when a tiny, obviously-surface edit scrambles the whole image, the lesson usually isn't "edit harder" or "phrase it more carefully." The lesson is that you picked a fragile seed, and no amount of prompt finesse will make a knife-edge stable. The move is to reroll — find a seed whose composition is robust — and do your iterating from there. A well-chosen seed forgives a lot of editing; a fragile one forgives none.

## Adherence and editability pull against each other

One more lever worth understanding, because it's the one people reach for when edits won't stick and it often makes things worse.

There's a setting that controls how literally the model obeys the prompt — how hard it's pushed toward the prompt and away from everything else. Turn it up and the model adheres more strictly: it carves deep, narrow basins around exactly what you asked for. Turn it down and adherence loosens: the basins get wide and shallow.

Deep narrow basins are exactly the fragile situation from the last section, systematized. Strong adherence means the smallest prompt change lands you in a *different* deep narrow basin — great obedience, terrible stability under editing. Loose adherence means edits deform the image gently instead of snapping it to a new arrangement — more forgiving to iterate on, but less faithful to any single prompt.

So there's a genuine tradeoff hiding in that one dial. You can have strict prompt-following or you can have graceful editability, but the same setting pushes them in opposite directions. If you're iterating and every tweak explodes the composition, cranking adherence up to force the change through is the wrong instinct — it makes the image *more* brittle, not less.

---

Strip away the mysticism and the seed does one honest job: it answers *where*. The prompt answers *what*. An edit stays local exactly when it doesn't pick a fight with the *where* — surface changes are safe, structural changes are not, and some seeds are too fragile to change at all.

Reproducibility lives in the seed. Editability lives somewhere else — in the attention pattern, in the geometry of the basin you happened to land in, in whether your change is high-frequency or low. Knowing which of those you're actually reaching for, and reaching for the right one, is most of the skill. The seed was never the version lock; it was just the trailhead.
