---
layout: ../../layouts/PostLayout.astro
title: "On-Device Is a Claim, Not an Architecture"
description: "Running inference locally is the beginning of a privacy guarantee, not the whole of one. Crash reports, backups, analytics, and the quiet assumption that an embedding is anonymous are where most of the data actually leaves."
date: 2026-08-05
tags: ["edge-ai", "privacy", "security", "architecture"]
---

*Part 10 of a ten-part series on the engineering problems that define edge AI.*

The strongest argument for running models on devices is not latency or cost. It is that the data never has to leave — and for systems built around personal information, that property is often the entire reason the product is permissible.

But "on-device" describes where a matrix multiplication happens. It does not, by itself, describe where the data ends up. A system can run every model locally and still leak comprehensively, through paths that have nothing to do with inference and that nobody designed. This last article is about the distance between the claim and the architecture that would justify it.

## Inference is one path among several

Start by enumerating where data in a device application actually goes. The list is longer than most teams expect, and the model is rarely the interesting entry.

**Crash reports** capture process state at the moment of failure. If a buffer of recent sensor data or a set of embeddings is live in memory, it can be included in a report that goes to a third-party service. This is the most common accidental leak in privacy-sensitive applications, and it is entirely invisible until someone reads a crash payload.

**Analytics** are added incrementally by people who are not thinking about the privacy model, often to answer a reasonable product question. Event properties accumulate, and a property that seemed innocuous — a count, an identifier, a duration — can be more revealing in aggregate than anyone considered when adding it.

**Device backups** copy application storage to a cloud service by default on major platforms. A local database of personal embeddings gets backed up to someone's cloud account unless it is explicitly excluded, and the default is inclusion.

**Logs** written during development frequently survive into production builds, and a log line printing a data structure for debugging becomes a log line printing personal data on a real user's device, where other processes and diagnostic tooling may read it.

**Model improvement pipelines** are the deliberate exception that quietly undoes the whole guarantee. A feature that uploads samples to improve accuracy is exactly the data flow the architecture was meant to prevent, and it is usually added later, by a different team, with a consent dialog that most users will not read carefully.

None of these are inference. All of them are exfiltration. A team that has verified "the model runs locally" and stopped has verified the least likely path.

## Embeddings are not anonymous

A belief worth dismantling early, because a great deal of otherwise careful design rests on it.

The reasoning is intuitive: the system does not store the audio or the image, only a vector of a few hundred numbers produced by a neural network. Numbers are not personal data. The original cannot be recovered from them.

The last clause is false, and it is false in a way that matters legally as well as technically. Embeddings are learned to preserve exactly the information that distinguishes individuals — that is their entire purpose — and research has repeatedly demonstrated that inputs can be partially reconstructed from embeddings, and that attributes not intentionally encoded can often be inferred from them. More decisively for practical purposes: an embedding *functions* as a biometric identifier. It can be matched against another sample to determine whether the same individual is present, which is precisely what identification means.

Under most modern privacy frameworks, that functional property is what determines the classification. A derived representation that can single out an individual is personal data, and in many jurisdictions biometric data specifically — a category carrying stricter consent, retention, and deletion obligations than ordinary personal information. The transformation into a vector is not anonymization. It is a change of format.

The practical consequence is that **an embedding store must be treated with the same care as the raw data it was derived from.** Encrypted, excluded from backups, genuinely deletable, and covered by whatever consent the raw capture would have required.

## The data nobody consented to

Here is the structural problem that distinguishes on-device recognition from most other privacy questions, and the one most likely to be overlooked by an engineer thinking in terms of user permissions.

A system that recognizes people holds data about **people who are not its users**. The user agreed to terms. The people the system recognizes did not, were not asked, and in most cases do not know the system exists.

Every framework built around user consent handles this badly, because its central mechanism — inform the user, obtain agreement — does not reach the affected party. The person whose representation is stored has no relationship with the product, no way to inspect what is held about them, and no mechanism to request deletion.

This is not a problem that better engineering solves. It is a constraint on what the product can defensibly be, and it is worth confronting during design rather than during legal review. The designs that hold up under scrutiny tend to share properties: the represented person participates knowingly in some way, or the data stays strictly on one device under the control of one person and never aggregates, or the context supplies genuine consent through some other channel — a workplace agreement, a clinical relationship, an explicit mutual exchange.

The designs that do not hold up are the ones where a system quietly accumulates identifiable representations of everyone a user encounters. That is not a compliance detail to be resolved later. It is the product's central question, and answering it late is expensive in a way no other item in this series is.

## Minimize before you protect

The most reliable protection for a piece of data is not collecting it. This sounds like a slogan and is a design method with concrete moves.

**Store the derived form, discard the source.** If the system needs to match embeddings, it does not need the audio or the images. Compute, store the vector, delete the raw capture immediately. This reduces both the sensitivity of what is held and the damage if it is exposed.

**Shorten every retention window that can be shortened.** A ring buffer holding a few seconds of sensor data is a fundamentally different risk from a rolling archive of the last month. Ask what the longest window is that the feature actually requires, then use that one rather than the one that was convenient.

**Aggregate at the earliest point possible.** If the product needs to know how often something occurred, store a counter rather than a list of occurrences with timestamps. The counter answers the question and cannot be mined for anything else.

**Delete on a schedule, not on request.** A system that removes data automatically after a defined period is more trustworthy than one that removes it when asked, because it does not depend on anyone remembering to ask.

## Deletion has to be real

Which brings back a point from Part 6, in its proper context.

Users delete things, and in many jurisdictions they have an enforceable right to have that deletion take effect. Implementations that mark records inactive and filter them from results — tombstoning, soft deletes, deferred compaction — are legitimate performance techniques and illegitimate privacy guarantees. The data still exists. It is recoverable. It appears in backups. It survives in whatever index or cache derived from it.

Real deletion means the bytes are gone, everywhere they were written: the primary store, any derived index, any cache, any backup, any log. Designing for this is much easier than retrofitting it, and it argues directly for the simplest possible storage — the flat array of Part 6 rather than a structure that defers removal for its own reasons.

It also argues for **encryption with per-record keys**, where the platform supports it. If each record is encrypted under its own key, discarding the key renders the record unrecoverable even if the bytes persist somewhere you cannot reach. This turns "delete everywhere" — a problem that is genuinely hard to guarantee across backups and replicas — into "delete one small key," which is tractable.

## Learning from a fleet without collecting from it

Systems still need to improve, and improvement usually seems to require data. There is a body of technique for getting most of the benefit without the collection.

**Federated evaluation** is the most immediately useful and the most overlooked. Rather than uploading samples to measure how the model performs, each device evaluates locally and reports only aggregate statistics — an accuracy figure, an error count. You learn how the system behaves across a population without any individual's data leaving their device. Most of what teams actually want from collected data is a summary statistic, and this delivers exactly that.

**Federated learning** extends the idea to training: devices compute model updates locally and a server combines them, never seeing the underlying data. It is more complex, and worth knowing that the updates themselves can leak information about the data that produced them, so it is usually paired with the next technique.

**Differential privacy** adds carefully calibrated noise to reported values, so that the aggregate remains accurate while no individual's contribution can be isolated. It provides a mathematical bound on what can be inferred about any one participant, and it costs some accuracy in exchange — a trade that is explicit and tunable rather than hoped for.

None of these are free, and all of them are cheaper than the alternative of holding a centralized store of personal data and accepting the obligations that come with it.

## Write down where the data goes

The discipline that ties this together is unglamorous and disproportionately effective: **maintain a written inventory of every piece of data the system touches.**

For each item, record what it is, where it is stored, how long it is kept, who can access it, whether it leaves the device and by what path, and how it is deleted. Cover the derived forms — embeddings, features, aggregates — not only the raw captures.

The value is not the document. It is that producing it forces someone to trace each path to its end, and that tracing is where the surprises live. Teams that do this routinely discover that a debugging log is still enabled in release builds, that a local database is included in cloud backups by default, that an analytics event carries a field nobody remembers adding, or that a crash reporter has been uploading process memory that contains exactly the data the architecture was designed to keep local.

None of those are exotic failures. They are the ordinary ones, and they are what actually undermines a privacy claim in a system where the model genuinely does run on the device.

## The reframing, and the end of the series

Running locally is an enabling condition. It makes a strong privacy guarantee *possible* by removing the largest and most obvious data flow. It does not make the guarantee true, because the data has other ways out and most of them were added by well-meaning people solving unrelated problems.

The guarantee becomes true when someone has traced every path, minimized what is held, made deletion real, and confronted the question of whose data this is and whether they had any say. That work is engineering — it just happens to be the kind that never appears in a benchmark.

Which is a reasonable note to end ten articles on. The recurring theme across all of them is that edge AI rewards the measurements nobody takes and the questions nobody asks: the gallery-size sweep, the partition report, the reliability diagram, the marginal contribution of each signal, the thirty-minute thermal plateau, the data inventory. The models are largely a solved problem you can download. The engineering is in knowing what to look at, and being willing to look when the answer is likely to be inconvenient.
