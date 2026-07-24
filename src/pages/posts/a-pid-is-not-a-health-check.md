---
layout: ../../layouts/PostLayout.astro
title: "A PID Is Not a Health Check"
description: "The most dangerous production failure isn't the crash — it's the process that passes every check you thought to run while doing exactly nothing. Liveness is a property of the whole system, not of any process in it."
date: 2026-07-23
tags: ["infrastructure", "reliability", "devops", "observability"]
---

The failures that teach people to distrust their monitoring are rarely the loud ones. A crash is loud: the process exits, the supervisor screams, a pager goes off, someone looks. The failure that erodes trust is the quiet one — the process that is *running*, supervised, restarting cleanly, logging no errors, and doing absolutely nothing useful. Every check you thought to run is green. The work is not getting done. And because nothing is technically broken, the gap can persist for hours or survive a reboot before anyone notices the outcome that was supposed to happen didn't.

This failure has a recognizable shape once you've seen it, and the shape generalizes far past any one tool. It comes from a single confusion that sits under most naive health checking: treating *the existence of a process* as evidence that *the process is doing its job.* Those are different claims, and the distance between them is where systems go dark while looking fine.

## What "running" actually proves

Strip a running process down to what its existence guarantees, and it's almost nothing. It proves the kernel has a scheduler entry with that identifier. It proves the program loaded far enough to not immediately exit. It does not prove the program parsed its configuration, connected to its dependencies, opened the socket it was supposed to open, subscribed to the queue it was supposed to drain, or is making any progress on the work it exists to do. A process can be alive and inert at the same time, and the operating system has no opinion about the difference — "doing the wrong thing forever" and "doing the right thing" are both just a process consuming a scheduler slot.

This is why "is the process up?" is such a seductive and such a weak health signal. It's trivially cheap to check, it's almost always available, and it answers a question adjacent to the one you actually care about without answering that question at all. Teams reach for it precisely because it's easy, and then quietly inherit its blind spot: it cannot distinguish a healthy worker from a hollow one.

## How a daemon ends up green but dead

The most common path into this state runs straight through the process supervisor — the very thing meant to keep you safe. A supervisor's contract is narrow and literal: *keep this exact command running, restart it if it exits, start it at boot.* It has no understanding of what the command is *for.* If the command it was told to run is subtly wrong — missing an argument, pointed at the wrong config, invoking a binary with no subcommand so it starts up and idles — the supervisor will faithfully keep that inert thing alive forever. It's doing its job perfectly. Its job just isn't the one you assumed.

This is worth dwelling on because it inverts the intuition. People add a supervisor to *increase* reliability, and it does — for the failure mode of "the process died." But it introduces a new, quieter failure mode: the supervisor will defend a broken configuration with exactly the same diligence it defends a working one. A crash loop, at least, is visible — the restart counter climbs. The truly dangerous case is when the wrong command happens to be a *stable* wrong command: it launches, it doesn't error, it doesn't exit, it simply doesn't do the work. Now you have a green supervisor status, a clean exit history, an empty error log, and a service that is functionally absent.

Automated installers and "set it up for me" commands are a frequent source of exactly this. A tool that generates its own service definition can generate a subtly incomplete one — a definition that runs the program but omits the instruction that makes the program do anything. The install succeeds. The service appears. Every surface-level check agrees it's fine. Only the outcome disagrees.

## Reading the signal that actually means something

The discipline that catches this is simple to state and easy to skip under pressure: **verify the process is doing the specific work, not merely that it exists.** In practice that means refusing to accept process-existence as an answer and insisting on three sharper questions.

First, *is it even the right process?* Inspect the actual command line the process is running, not just its name. Two invocations of the same binary can share a name and differ in the one argument that separates "serving traffic" from "idling." The name tells you almost nothing; the arguments tell you what it's actually trying to do.

Second, *whose process is it?* A background service adopted by the system's init has a different lineage than a copy you started by hand in a terminal. Confusing the two is how people verify a hand-started instance, conclude the durable one works, and get surprised at the next reboot. Lineage is checkable, and it disambiguates "this survives a restart" from "this is alive because I'm still logged in."

Third, and most important, *is the work happening?* Every daemon worth running emits some application-level signal that it's actually functioning: connections registered, requests accepted, a queue depth that moves, a heartbeat advancing. That signal — not the process's existence — is the real health check. A connector should be asked "are your connections established?" A worker should be asked "is the backlog draining?" A server should be asked "did a real request just get a real response?" These are the questions whose answers correlate with the outcome you care about. The process being scheduled does not.

There's a small, recurring trap inside that third question worth naming, because it fools careful people: the log stream. Many long-running programs write their informational output — including the very lines that would confirm they're working — to standard error, not standard output, reserving each stream for a different purpose. Look in the wrong stream and you find an empty file, and an empty log is dangerously ambiguous. It can mean "nothing happened," or it can mean "you're reading the wrong place." Read the empty stream and you can talk yourself into believing a perfectly healthy service is dead, or a dead one healthy, with equal confidence. Knowing which stream your tools speak into is not trivia; it's the difference between a diagnosis and a guess.

## The decoupling underneath

Zoom out from the individual daemon and the same class of bug appears at the level of the whole deployment. The green-but-dead daemon is one instance of a more general hazard: **"deployed" and "reachable" were allowed to be independently true.** The application shipped, the containers came up, every local check passed — and the thing that actually connects the running application to the outside world was a separate component that failed, or was never wired, or quietly stopped. The deploy succeeded. The system was still dark.

Any time reachability depends on a component that lives *outside* your unit of deployment — an ingress controller, a load-balancer registration, a DNS change, a tunnel or connector, a service-mesh sidecar — you have a latent decoupling bug waiting for the day those two truths diverge. The deploy will report success against the thing it deployed while the thing that makes it reachable sits in a different state entirely.

Two defenses work, and mature teams use both. The first is to *collapse the two truths into one event*: make the reachability component part of the deploy artifact itself, so that bringing the application up necessarily brings its path to the world up in the same motion, with no separate step to forget or misconfigure. When "deployed" and "reachable" are the same operation, they can't disagree. The second is to *health-check the outcome, end to end*: probe the service the way a real user reaches it — through the public edge, across the actual network path — not by asking a local port whether it's listening. A check that hits the process on the loopback interface will stay green through this entire failure, because the process really is listening; it's everything between that port and the user that's broken. Only the end-to-end probe crosses the gap where the failure lives.

## Cutover discipline

There's a corollary for the moment you *fix* one of these, because the fix is itself a small, sharp reliability exercise. When a working-but-temporary thing is currently carrying load and you're replacing it with the permanent version, the rule is: **never retire the old path until the new one has proven it is carrying its own load.** Not "until the new one started" — until it has demonstrably taken over the work, by its own application-level signal, independently of the old one.

Two identical instances running side by side is usually safe — redundant paths to the same backend simply share the load, with no correctness hazard, precisely because they're interchangeable. That overlap is a feature: it buys you a window to confirm the replacement is genuinely live before you tear down the incumbent, with zero gap in service. And when you finally do remove the old one, remove it *precisely* — target the specific instance, not a pattern that might match both. The failure mode of a careless teardown is killing the replacement along with the thing you meant to retire, turning a clean cutover into the outage you were trying to avoid.

## The meta-lesson

Underneath all of this is one idea, and it's older than any particular technology: **monitor outcomes, not proxies.** "Is the process running" is a proxy. "Is the disk mounted," "is the port open," "did the container start" — all proxies. Each is cheap, each is available, and each answers a question that merely lives near the one you care about. The outcome — is the work getting done, is the user being served, is the queue draining, did the request succeed — is more expensive to check and is the only thing that actually correlates with the system being healthy.

The reason green-but-dead failures are so corrosive is that they exploit exactly this gap. They keep every proxy green while the outcome quietly fails, and a monitoring strategy built on proxies has, by construction, nothing to say about them. The fix isn't more proxies or more alerting on process liveness. It's moving your health checks down to the outcome — asking the daemon to prove it's doing its job, asking the deployment to prove it's reachable the way a user reaches it — and treating the existence of a process as what it actually is: a necessary condition, and nowhere close to a sufficient one.
