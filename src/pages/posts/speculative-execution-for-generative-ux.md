---
layout: ../../layouts/PostLayout.astro
title: "Speculative Execution for Generative AI UX"
description: "GPU cold starts make the first image the slowest one — right when the user is judging your product. Instead of paying for an always-warm GPU, start the job before they click Generate, and treat the money problem with a payments-industry trick."
date: 2026-07-09
tags: ["ai", "image-generation", "ux", "billing", "infrastructure"]
---

The slowest image your product will ever generate is the first one. GPU providers deallocate idle instances, so after a quiet stretch the next request pays a cold start — 15 to 30 seconds of model weights loading before a single denoising step runs. Every image after that is fast, because the GPU is warm. But the user isn't judging your product on image seven. They're judging it on image one, and image one is the one that hangs.

I hit this building a manga generator: a user writes a story, reviews a storyboard, clicks Generate, and panels render one at a time. Panel one routinely took three to five times longer than the rest. Same model, same prompt complexity — just a cold GPU.

The standard menu of fixes is unappetizing at small scale:

- **Keep an instance always warm.** For the A40-class GPU my image model runs on, that's about $21/day — roughly $648/month — to hold a machine idle so that panel one is fast. The break-even math said I'd need thousands of avoided cold starts a day to justify it.
- **Keep-warm pings.** A cron job runs a cheap inference every few minutes. Cheaper (~$0.50–$2/day), but it's real spend on fake work, the cadence never quite matches when users actually show up, and now there's a scheduled job whose whole purpose is generating images nobody sees.
- **Eat it.** Show a spinner and some copy about warming up. This is what most products do, and it's why most generative products feel broken for the first thirty seconds.

There's a fourth option, and it comes from an old hardware idea: don't wait for the instruction — execute it speculatively and throw the work away if you guessed wrong.

## The intent signal was already there

Right before Generate, my users sit on a Review screen reading their storyboard. Typically for twenty or thirty seconds — almost exactly the length of a cold start. And a user on the Review screen has essentially already decided. They wrote the story, placed the characters, and walked through three steps of a wizard to get here. The click is a formality.

So: when the Review screen settles (a 1.5-second debounce, so a quick back-click doesn't trigger it), the app quietly starts the real job. The GPU spins up and renders panel one while the user is still reading. When they click Generate, the app doesn't start anything — it *attaches* to the job that's already running. Panel one is either done or seconds away. The cold start didn't go away; it got hidden inside time the user was going to spend anyway.

One accident of architecture made this almost free to build: my pipeline already pauses after every panel to wait for the user's approve/redo decision. A speculatively started job renders panel one and then just… parks, in a wait state that already existed. "Generate" became a `confirm` signal pushed onto the same queue the approval buttons already use. I didn't build a speculation engine. I moved the start time.

## The hard part is money, not latency

Here's what makes generative speculation different from CPU speculation: mispredicted instructions cost nothing, but a mispredicted image render costs real money — my panel one costs about $0.04 on the GPU. And it's the *user's* money too, if your product runs on credits. Speculating with someone else's balance is a good way to lose their trust the first time it goes wrong.

The naive designs both fail:

- **Charge at speculative start, refund on abandon.** A user who glances at Review and closes the tab got charged for nothing, and now you're writing refund code — and a refund that silently fails (a crash between render and refund) is silent theft.
- **Charge at the Generate click.** Then the speculative render happens before any charge, and a user can farm free images by visiting Review and walking away — or worse, polling the job's result endpoint directly, since the job record is theirs.

The payments industry solved this decades ago: **authorize, then capture.** At speculative start, the app places a *hold* on one credit — a single atomic database decrement, so two racing tabs can't both win the last credit. The balance visibly dips; nothing is spent. Clicking Generate eventually *captures* the hold on the normal success path. Abandoning *releases* it. And crucially, a crash leaves a **visible stale hold** that a periodic sweep can release — the failure mode is "reconcilable ledger entry," not "money quietly gone."

The hold does double duty as the eligibility gate: speculation only fires for users who actually have a credit. A broke user's speculative job fails the hold instantly — no GPU touched, no spend — and their Review screen stays deliberately silent. They find out at the Generate click, where a purchase prompt makes sense. Speculation is a perk for people who can pay, invisible to people who can't.

## The failure mode nobody thinks about: your own worker pool

My worker runs four concurrent jobs, each with a two-hour timeout (the pipeline has a human-in-the-loop approval step, so jobs legitimately live a long time). Do the math on naive speculation: four users visit the Review screen, wander off, and their abandoned speculative jobs sit in the approval-wait state holding all four slots — for up to two hours. The entire product is frozen by window shoppers.

So unconfirmed speculative jobs get their own clock: no confirmation within ten minutes and the job cancels itself, releases its hold, and frees the slot. Back-navigation cancels explicitly and immediately. The bound lives inside the job's wait loop, not in the worker's global timeout — confirmed jobs still get their two hours.

There was one more integration seam: the user attaches to a job that started before they were listening. My progress events ran over a Redis pub/sub channel, and pub/sub has no memory — attach late and you've missed everything. The fix was a replay buffer: every event also gets appended to a Redis list with a TTL, and the SSE endpoint replays the backlog before streaming live. Attach at any point in the job's life and you reconstruct the full picture.

## The bug that proved the whole approach needed a browser

Everything above passed its tests. Then a real person with a deliberately emptied account walked through the flow and saw "Preparing your first panel…" on the Review screen — the speculation hint — while having zero credits.

The submit endpoint returns 202 the moment the job is queued. The credit hold happens a beat later, *inside the worker*. My UI keyed the hint off the 202, so a broke user's job would enqueue, flash the hint, and die with an insufficient-credits event a hundred milliseconds later — with nobody listening, because the whole point of the design was not opening an event stream at Review. The hint lied.

The fix was small (poll the job's status once, two seconds after submit, and only then show the hint). The lesson wasn't small: **speculative UX moves your async boundaries around, and every claim your UI makes about background work needs to be re-derived from what the backend has actually confirmed** — not from "the request was accepted." Accepted and eligible are different states, and the gap between them is exactly where speculation lives.

## The general pattern

None of this is manga-specific. It applies anywhere a generative product has a compose-or-review step in front of an expensive render — prompt review before a video generation, voice selection before TTS synthesis, a settings screen before a fine-tune kicks off. The checklist that made it work:

1. **A real intent signal.** Speculate on "reached the review screen after three steps of investment," not on "loaded the homepage." Your misprediction rate is your waste rate.
2. **Dwell time that covers the latency.** The user needs something legitimate to do while the speculation runs. Reading their own storyboard qualifies.
3. **Reversible money.** Hold-then-capture, atomic at the decrement, released on every non-success path, reconcilable when release fails. Never charge-then-refund.
4. **Bounded resource occupancy.** Speculative work must hold scarce resources (GPU slots, queue workers) on a short leash with its own timeout — misprediction should cost cents, never capacity.
5. **Silence for the ineligible.** Users who can't proceed shouldn't see speculation exist. Gate at the cheapest possible point, before any expensive work.

The economics ended up almost embarrassing. Worst case, an abandoned speculation costs me one panel — four cents — in exchange for the product's most important moment feeling instant. The always-warm alternative costs $648 a month whether anyone shows up or not. Speculation only spends money when a probably-paying user is actively about to spend more of it. The branch predictor was right all along: don't pay to be ready — pay to be *probably* right, and make wrong cheap.
