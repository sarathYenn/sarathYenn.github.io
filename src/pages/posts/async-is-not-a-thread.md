---
layout: ../../layouts/PostLayout.astro
title: "Async Is Not a Thread, and Other Guardrails for Edge AI"
description: "An AI feature on a phone mixes cheap local work that must never freeze the screen, slow paid calls to a model, and tools whose answers matter more than the model's. Seven guardrails keep the three from leaking into each other."
date: 2026-11-01
tags: ["edge-ai", "concurrency", "reliability", "architecture"]
---

Most AI features that run partly on a device and partly in the cloud are built from three kinds of work. There is cheap local work that runs all the time and must never freeze the interface. There are slow, paid network calls to a large model. And there is a source of truth that is not the model at all: an engine, a registry, a policy. Most of the failures in these systems come from treating the three as if they were the same kind of thing.

Imagine an anti-scam assistant on a phone. A small on-device model screens incoming messages and calls all day, for free. When something looks risky, or the user taps "is this a scam?", the app sends evidence to the cloud, where a large model explains and deterministic tools check the facts: a link reputation service, a list of known numbers, a lookup of payment references.

The same shape appears elsewhere. A digital identity app runs liveness checks and document capture on the device continuously, and sends the expensive face match and document verification to the cloud only at submission. A VPN or secure-access client classifies traffic and applies policy on the device per flow, and asks the cloud for threat intelligence on demand. Always-on local work, occasional slow and paid cloud calls, and a truth that the model does not get to decide. What follows are seven guardrails for keeping those apart.

## 1. Async is not off-thread

Marking a function as asynchronous — `suspend` in Kotlin, `async` in JavaScript, Swift or Python — lets it *wait* without blocking. It does not move its CPU work anywhere. A coroutine launched from a screen's scope runs on the main dispatcher; a JavaScript async function runs on the page's only thread. Between two suspension points, the code holds that thread exclusively.

The trap is that the keyword looks like isolation. A function that declares itself asynchronous and then runs a 40-millisecond classifier inline still freezes the interface for 40 milliseconds; the declaration changed the return type and nothing else. In Kotlin terms, making a function `suspend` does not change which thread it runs on — only an explicit switch to a background dispatcher does. In JavaScript there is no such switch at all: the only way to get another thread is a separate program, a worker, that you talk to by sending messages.

The guardrail is a budget. Give every recurring local job a time allowance on the interface thread — a 60 Hz frame is about 16 milliseconds — and measure it. Anything over budget moves off the thread or runs less often. For the anti-scam app, screening a long message thread with an on-device model can easily take tens of milliseconds, which puts it firmly on a background dispatcher, or on a dedicated single-thread one if the model runtime is not thread-safe.

## 2. Isolation has a price

In Kotlin, background threads share memory, so handing a bitmap to a background dispatcher is nearly free. Many isolation boundaries are not like that, and the cost of crossing them is easy to forget.

| Boundary | Shares memory? | What crossing costs |
| --- | --- | --- |
| Coroutine dispatcher switch | Yes | A thread hop |
| Web worker, or a browser extension's hidden page | No | A copy of the data, or a transfer of its buffer |
| Separate OS process (an engine, an inference daemon) | No | Serialisation plus inter-process communication |
| Isolated service on a mobile OS | No | An IPC call and its size limits |
| Device to cloud | No | Encoding, upload, latency, and money |

The guardrail is to cross a boundary per *decision*, not per *frame*. Send a compact input — a crop, a compressed image, a feature vector — and receive a small answer. A job that runs twice a second and takes 25 milliseconds may well cost more to ship across a boundary than to run in place, while a 300-millisecond search that runs only when something changes is an easy win to move. Measure the round trip with a realistic payload before deciding.

Platform rules choose the boundary as often as performance does. A browser page with a strict security policy can refuse to start a worker at all; a mobile OS can kill a background service that holds too much memory. For an identity app, this is the camera pipeline: liveness runs in-process on downscaled frames, and only the final capture crosses into the secure verification path.

## 3. Drop, don't queue, and re-check after every await

For a stream of frames, packets or messages, an answer about an old input is worth very little. If work arrives faster than it finishes, a queue only grows latency and memory. Keep **one job in flight per kind of work**, and skip whatever arrives in the meantime — conflate the stream so a slow consumer only ever sees the latest item.

In outline:

    on new frame:
        if a scan is already running: skip this frame
        otherwise: start a scan in the background

The second half matters as much. **While a coroutine is suspended, the world moves on.** The user leaves the screen, the call ends, the VPN reconnects. When the result comes back, check that it still applies before acting on it — tag each job with a session or generation number, and drop answers that belong to a world that no longer exists. On a single thread this needs no locks: between two suspension points nothing else runs, so a plain flag is enough.

For the anti-scam app, a verdict about a call that has already ended should update the call log, not pop a warning over whatever the user is doing now.

## 4. One deadline per request, checked before every paid step

Per-call timeouts are not enough, because each hop sets its own and they never add up to anything the user agreed to. Consider a plausible chain: the client gives up after 90 seconds; on the server, a vision read is allowed 60 seconds, an engine analysis a couple of seconds, and an explanation 120. In the worst case the server keeps working — and paying — for three minutes on a request the client abandoned after one and a half.

The fix is a single **deadline**, set where the request starts and carried through every hop. Google's SRE book calls this deadline propagation, and several RPC frameworks can do it automatically. Before each slow or paid step, the code asks whether there is still time, and if not, it stops.

There is a subtlety that catches experienced engineers: **timeouts do not stop CPU work.** Kotlin's timeouts cancel cooperatively — a tight loop that never checks for cancellation runs to the end. JavaScript's abort signals cancel a network wait, not a running function. A timeout around on-device inference therefore means "stop waiting and move on", not "the work has stopped". Long local loops need explicit cancellation checks, and late results must be ignored when they arrive (guardrail 3).

For the anti-scam app, the user is mid-call. A two-second deadline for "warn or don't" beats a perfect answer at twenty seconds, which means the fallback — "couldn't check in time; be careful with payment requests" — is designed before the happy path.

## 5. No silent retries on paid calls; reserve budget before spending

Client libraries often retry by default. For a free, idempotent read, that is helpful. For a model call billed per token, a silent retry is silent extra spend, and during an outage it multiplies load at exactly the moment the provider is struggling.

- **Retries off by default on paid calls.** Retry only where the caller decides to, with backoff and jitter, and only inside the request's deadline.
- **Reserve, then spend.** Take a unit from the user's or session's budget *before* the call, with an atomic increment that is backed out if it overshoots the cap, so concurrent requests cannot exceed it together.
- **Spend only on intent.** Cloud model calls fire on a user action or a clear local trigger, never from a background loop.
- **Fail cheap first.** Order each request so the free checks — schema, input validity, whether the downstream engine is up — run before the paid step.

For the anti-scam app, on-device screening is free and always on, while the cloud explainer is capped per user per day. A flood of suspicious messages, which is itself a common scam pattern, must not become a flood of paid calls; it gets batched or summarised locally first.

## 6. A model's output never reaches a tool or a user unchecked

Large models are fluent and can be confidently wrong, and their own stated confidence is not a reliable signal. A pattern that shows up in vision tasks is instructive: a model reading a structured image will occasionally swap two categories — a light item read as dark, a digit read as its neighbour — *without* listing it as uncertain. If that output feeds a deterministic tool, the tool faithfully computes the right answer to the wrong question, and the explanation built on top of it is confidently wrong as well.

Two layers of checking address this.

**Strict schemas at every boundary.** Reject unknown fields rather than ignoring them, bound every string and list, use enumerations for fixed choices, and generate client types from the schema so the two sides cannot drift. Then check *meaning*, not just shape: a well-formed record that describes an impossible state is still rejected.

**An independent verifier before any tool acts.** Compare the model's output against a cheap signal the model cannot influence — usually something the device measured itself. If they disagree, the user gets an honest "couldn't verify this" instead of a confident wrong answer. The model proposes; a deterministic system with its own evidence disposes.

| Domain | The model proposes | The independent check before acting |
| --- | --- | --- |
| Anti-scam | "This message is from your bank" | Sender and links checked against a verified registry; the model never marks a sender trusted on its own |
| Digital identity | Fields read from a document photo | Machine-readable zone and chip data, checksums, issuer format rules |
| VPN and secure access | "Allow this destination" | The policy engine and threat intelligence decide; the model only explains |

The rule underneath all three: **the source of truth is never fed unchecked input.**

## 7. Routing is a policy, not a fallback chain

A common design is a cascade: try on the device; if unsure, try the cloud; if that fails, try a bigger model. It is hard to reason about, because cost, latency and privacy all end up depending on runtime luck. Decide the split in advance, per kind of work.

| Runs on the device, always | Goes to the cloud, only on intent |
| --- | --- |
| Cheap, frequent, latency-critical work: detection, screening, classification | Expensive, rare, accuracy-critical work: deep reads, explanations, registry lookups |
| Works offline, private by default | User-triggered or clearly rate-limited, capped per user |
| Its output is a signal, often used to check the cloud's answer | Its output is checked before use (guardrail 6) |

Two further rules follow. **The device's cheap signal is a feature, not a fallback.** An on-device classifier's view of a message is exactly the independent evidence that can question a cloud model's "this is safe". And **when the cloud fails, say so plainly.** "Couldn't check this message right now" is honest; a silent downgrade to a weaker path that looks identical to the user is not.

Privacy belongs in the routing decision too. For identity and anti-scam applications, decide which data may ever leave the device — a hash, a link, a cropped region — before choosing any model.

## A checklist

| Guardrail | The failure it prevents | How to test it |
| --- | --- | --- |
| A thread budget for every recurring local job | Jank, unresponsive-app reports, frozen pages | Measure each job's time per run on a slow device in CI |
| Cross boundaries per decision, not per frame | A copy that costs more than the work it saves | Time the round trip with a realistic payload before moving work |
| One job in flight; stale results dropped | Growing latency; warnings about the wrong call or screen | Feed inputs faster than they are processed; change state mid-job |
| One deadline through every hop | Paying for answers nobody sees; hung requests | Make the client give up early and assert the server skips the next paid step |
| No silent retries; budget reserved first | Surprise bills and retry storms | Send concurrent requests at the cap and confirm it is never exceeded |
| Strict schemas plus an independent verifier | Confidently wrong output reaching a tool or a user | Flip one field of a model answer and confirm it is refused |
| Routing decided up front; failures stated plainly | Unbounded cost, data leaving the device, silent downgrades | List every cloud call and its trigger; cut the network and read the message |

None of these guardrails is novel on its own. Most are decades old in distributed systems. What is new is how many of them a single AI feature on a single phone now needs at once — and how easy each is to miss when the code that violates it still compiles, still passes its tests, and still feels asynchronous.

## Further reading

- [What Color is Your Function?](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) — Bob Nystrom on how async splits a codebase in two.
- [Best practices for coroutines in Android](https://developer.android.com/kotlin/coroutines/coroutines-best-practices) — main-safe suspend functions and injected dispatchers.
- [ANRs](https://developer.android.com/topic/performance/vitals/anr) — what a blocked main thread costs on Android.
- [Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/) — Jake Archibald on how a browser's single thread orders work.
- [When should you be using Web Workers?](https://surma.dev/things/when-workers/) — Surma on the main-thread budget on slow phones.
- [Concurrency and async / await](https://fastapi.tiangolo.com/async/) — why blocking calls belong on a thread pool, not the event loop.
- [Timeouts, retries, and backoff with jitter](https://builder.aws.com/content/3EumjoZascWd1oZiEgL8ORlv3qE/timeouts-retries-and-backoff-with-jitter) — Marc Brooker, Amazon Builders' Library.
- [Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) — the Google SRE book on deadline propagation and retry budgets.
- [Deadlines](https://grpc.io/docs/guides/deadlines/) — gRPC's guide to setting and propagating them.
- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/) — especially Improper Output Handling, Excessive Agency and Unbounded Consumption.
- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — Anthropic on tool design and guardrails for model-driven tool use.
