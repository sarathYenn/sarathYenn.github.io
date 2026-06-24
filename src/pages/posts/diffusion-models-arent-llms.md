---
layout: ../../layouts/PostLayout.astro
title: "Diffusion Models Aren't LLMs"
description: "I tried LLM prompt-engineering tricks on a diffusion model. Most of them backfired. Here are the counter-intuitive lessons that finally got the model to render what I asked for."
date: 2026-06-23
tags: ["ai", "diffusion-models", "prompt-engineering"]
---

I've spent time recently working with diffusion image models — the Qwen / Flux / SD3 family of modern transformer-based text-to-image systems. Coming from a year of LLM prompt-engineering, I assumed my instincts would carry over. They mostly didn't.

Here's a list of things that surprised me, ordered roughly by how much they cost me to figure out.

## 1. Captions beat prose, by a wide margin

I'd been writing image prompts as descriptive prose:

> *"A woman stands at the counter, her eyes closed in pleasure as she sips her coffee, her expression momentarily blissful. The golden morning light streams through the window behind her, casting warm tones across her face. The mood is contemplative and tinged with quiet satisfaction."*

Felt like good writing. The renders were mediocre — backgrounds were vague, the subject felt anonymous, the "mood" never came through.

I rewrote the same prompt as a caption:

> *"Medium close-up, kitchen interior, golden morning light from window stage-right. Woman at right foreground, blissful face, coffee cup held in both hands. Counter and houseplant in soft background. Cel-shaded illustration, soft warm palette."*

Sharper composition, better lighting, more recognizable subject. Why?

Diffusion models are trained on **captioned images**. The captions look like the second example, not the first. When you feed prose, you're asking the model to render something it has never seen at train time — narrative text — into something it knows how to produce. The model translates twice and loses fidelity each step.

A caption is dense visual tokens in the order the model expects: *shot type, lighting, subjects with positions, props, style anchor.* Every token earns its keep. Prose carries words like *"moment", "captures", "tinged with"* that have no pixels.

**The mental model shift:** stop writing for a human reader. Write for a captioner.

## 2. "No X" actually summons X

I had a recurring problem where extra figures kept appearing in scenes that were meant to feature only one character. I added to the prompt:

> *"No other people in this scene. No second character. No extra figures in the background."*

Extra figures appeared **more** often after I added that.

This is the classic *don't think of a pink elephant* problem. Diffusion text encoders don't strongly handle negation — they tokenize and weight. The token "people" / "character" / "figures" floats in the prompt context regardless of the word "no" preceding it. The model renders them.

The fix is the **`negative_prompt` parameter**, which most diffusion models on inference endpoints expose but you have to actually use. I'd been ignoring it and trying to negate inside the positive prompt. Once I moved exclusions to `negative_prompt` (*"extra people, additional figures, background characters"*), they actually got excluded. The architectural mechanism is classifier-free guidance: the model is told *"render toward positive, away from negative"* as separate signal vectors. They're processed differently from the inside.

**Lesson:** anywhere you find yourself writing *"no X"* in a positive prompt, you have the wrong tool. Use `negative_prompt`.

## 3. Position in the prompt matters more than presence

For a long stretch, a character I was describing as a woman kept rendering as a man.

The character description was thorough — multiple sentences, explicitly female. The model still drew a man.

I traced through the assembled prompt. The character description was at character position ~2200. The scene description started at position 0 and mentioned the character by name with no gender word adjacent. By the time the model processed *"woman"* deep in the description block, it had already started composing the image based on the scene context. Diffusion text encoders front-load attention — earlier text anchors composition more strongly. The character description was supporting context, processed too late to override what the scene committed to.

The name didn't help either. Names that are unambiguous in their cultural context can be ambiguous to the model's training distribution, especially across languages and regions. If the scene doesn't include a gender anchor next to the name, the model guesses, and it often guesses wrong.

**Fix:** inject the gender / role inline in the scene at first mention. The scene became *"<name> (the mother) listens attentively..."* — the gendered word now appeared at position ~90 instead of ~2200. Rendered correctly from the next generation onward.

**Lesson:** tokens in the topic sentence count more than tokens in supporting context. Don't bury gender / role / spatial anchors deep in the prompt where the model will already have committed by the time it gets to them.

## 4. Pronouns smear across characters

Multi-character compositions are uniquely brittle in ways single-character ones never are.

Consider a two-character scene where one character has a distinctive physical state — say, one is soaked from rain, the other is dry. If the prompt describes the soaked character with *"water dripping from his clothes"*, the model often renders **both** characters as soaked.

The culprit is the pronoun *"his"*. To a human reader, *"his clothes"* inside the soaked character's description unambiguously refers to him. To a diffusion text encoder, *"his"* is just an unscoped possessive token in a prompt that has two male characters. There's no co-reference resolution at the model level — the concept "water on clothes" floats globally and gets applied to whichever male the model is rendering at that latent step.

**Fix:** programmatically replace pronouns with the character's name before sending the prompt. *"Water dripping from Alex's clothes."* Verbose to read, but the model now has explicit token-level attribution. Apply it to every character bible: every `he/she/his/her/they/their` becomes the character's actual name.

**Lesson:** diffusion text encoders don't resolve coreference. If a trait must scope to a specific character, name them every time. Repetition costs prompt budget but stops attribute leakage.

## 5. Motion verbs cause character duplication

This one shocked me. A prompt described a character as *"standing at left foreground entering doorway."* The model rendered the same character **twice** — once at the doorway, once already in the foreground.

The verb *entering* implies motion across a threshold — a transition from outside to inside. To a diffusion model rendering a single frozen frame, *"X is at position A and X is moving to position B"* gets resolved by drawing X at both A and B. The model can't render time-as-a-process; it renders state. Two implied states collapse into two figures.

This generalizes to every verb implying position change: *entering, exiting, approaching, walking, crossing, climbing, descending, leaving, returning.* All can trigger duplicate rendering. Static framings (*"standing in the doorway," "seated at the table," "at the threshold of"*) don't.

**Lesson:** diffusion is a frozen frame. Never describe transitions. Pick one static position per subject and stick with it.

## 6. Logical contradictions render both interpretations

Consider a prompt fragment like:

> *"The subject stands alone at the counter... In the softer background, a second figure is visible heading toward the doorway."*

The first clause says *alone*. The second introduces a second figure. A human reader skims the contradiction — *"the subject is the focus, the second figure is contextual background."* The diffusion model renders both interpretations:
- *Alone* → composition with the subject in focus, surrounded by empty space.
- *A second figure is visible...* → a second character in the background.

What you get: the second figure appears in the background **and** the empty space next to the subject gets filled by... a duplicate of someone (often the wrong character entirely), because the prompt insisted there was a second character to render and the model couldn't reconcile *"alone"* with *"the other is also here."*

Visible logical contradictions don't get reasoned about. They render out as ambiguity, which usually means rendering both interpretations simultaneously.

**Lesson:** every clause about composition must be internally consistent. There's no model-side reasoning to resolve *"alone but also someone else is there."*

## 7. Training corpus priors are stronger than your instructions

A scene with two children in a domestic interior — emotional moment, soft lighting. The model rendered the two children **plus a phantom adult woman watching from the doorway.**

Nothing in the prompt mentioned an adult woman. The negative prompt said *"no extra people."* No effect.

This is training corpus prior winning. Most image-training corpora have thousands of *"young children in domestic interior, emotional scene"* compositions where a parent figure is present in the frame. The model has a strong attractor for that composition. Mild negative prompt language doesn't fully override a deeply-trained prior.

You can fight it three ways: more specific negative phrasing (*"no mother figure, no adult woman, no parent"*), higher classifier-free guidance scale to force literal adherence to your prompt, or per-attempt redo when it slips through. Sometimes the answer is just *"redo this one."*

**Lesson:** the model has opinions baked in from training. You can pressure them but not always override them.

## The meta-lesson

LLM prompt engineering is fundamentally **communication with a system that can reason**. You explain what you want, give examples, set constraints, and the model follows along. Polite framing, structured sections, and explicit instructions all help.

Diffusion prompting is **communication with a system that pattern-matches against training data**. You're not explaining; you're aiming. The shape of the prompt — vocabulary, order, density, which channel (positive vs negative) — matters as much as the content.

A lot of standard "good prompt engineering" advice for LLMs actively hurts diffusion output. *Use clear section headers* — labels often leak as rendered text in the image. *Add narrative context* — context becomes pixels you didn't want. *Explain what to avoid* — you summon what you tried to exclude. *Be verbose for clarity* — verbosity dilutes the per-token attention.

Diffusion wants the opposite: dense visual tokens, explicit positions, training-corpus-shaped captions, no narration, no pronouns, no contradictions, no transitions.

If you're getting bad diffusion output and your instinct is to *engineer* the prompt the way you would for an LLM — adding context, structure, careful instructions — consider that your prompt is probably **too engineered**. Try a 280-character caption that reads like a tag list. The output is often dramatically better, immediately.

---

The biggest single quality jump I saw came from rewriting prompts from "describe the scene like a painting in 3-5 sentences" to "output a diffusion-model caption in 250-350 dense visual tokens." Prose to captions. A few hours of work, weeks of compounding pain avoided.

The model was always going to render what its training data told it to render. The job isn't to teach it; it's to learn what shape of request it actually understands.
