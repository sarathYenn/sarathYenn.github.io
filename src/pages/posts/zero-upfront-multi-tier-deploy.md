---
layout: ../../layouts/PostLayout.astro
title: "Zero-Upfront Multi-Tier Deploy: Prod + Staging on a Mac Mini Behind One Cloudflare Tunnel"
description: "Environment isolation isn't a hosting-tier problem you solve by opening an AWS account on day one — it's a configuration property. How a bootstrapped product runs a real prod and staging tier side by side on one Mac Mini behind a single Cloudflare Tunnel, and when that stops being the right answer."
date: 2026-07-17
tags: ["infrastructure", "bootstrapping", "devops"]
---

The day I needed a staging environment, my first instinct was to open a second cloud account. Not because I'd thought it through — because that's the reflex. Prod and staging are different tiers, different tiers mean different infrastructure, different infrastructure means a second VM, a second bill, maybe a second region for good measure. I had the AWS console open before I'd asked myself whether any of that was actually necessary.

It wasn't. I was building [genkos.app](https://genkos.app), a small pipeline that turns text stories into illustrated manga panels, and at the point I needed staging I had zero paying users and one Mac Mini sitting under my desk doing nothing after business hours. The thesis that stopped me from opening that second account: **isolation feels like a hosting-tier purchase, but it's a configuration property.** Two environments are isolated when nothing about how they run assumes the other doesn't exist — not when they're billed separately. I ended up running a real prod and a real staging tier, both with their own database, their own cache, their own images, on that same Mac Mini, at zero marginal cost, and the distinction held up under actual use for months.

## The reframe

Here's the claim stripped of the specific hardware: "two VMs, two accounts, two bills" and "two compose projects on one box" are *equally isolated*, provided nothing in either stack assumes it's the only tenant on the machine. Isolation is a property of the boundary you draw, not of the number of physical machines behind it. A boundary drawn with distinct ports, distinct named volumes, distinct database names, and distinct image tags is just as real as a boundary drawn with a VPC and a separate AWS account — it's enforced by different mechanisms, but a crossed wire is a crossed wire either way, and the failure mode you're actually defending against (staging traffic hitting the prod database, a staging migration clobbering prod data) is prevented by both.

The one constraint that has to hold for this to work is stricter than it sounds: every environment-specific value — hostnames, credentials, price IDs, feature flags — has to flow in through environment variables at deploy time, never get baked into application code or committed as a literal. The moment a config value gets hardcoded "just for now," the two tiers stop being interchangeable deployments of the same code and start being two divergent codebases that happen to share a git history. That single rule is what makes running both tiers on one box safe instead of reckless.

## The concrete topology

In practice this is two independent, non-overlapping Docker Compose projects on one machine, brought up with their own project name and their own compose file:

```
docker compose -p genkos-prod    -f compose.prod.yml    --env-file compose.prod.env    up -d
docker compose -p genkos-staging -f compose.staging.yml --env-file compose.staging.env up -d
```

Nothing is shared between them except the host. Every service gets a distinct port, a distinct named volume, and a distinct image tag:

| | prod | staging |
|---|---|---|
| API port | 8000 | 8001 |
| Postgres port | 5433 | 5434 |
| Redis port | 6379 | 6380 |
| Web port | 3000 | 3001 |
| Database | `manga_prod` | `manga_staging` |
| Volumes | `postgres_prod`, `cache_prod` | `postgres_staging`, `cache_staging` |
| Images | `manga-svc:prod`, `manga-web:prod` | `manga-svc:staging`, `manga-web:staging` |

A trimmed compose excerpt for one tier's database and API, with the placeholder pattern the real files follow:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: manga_prod
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD_PROD}
      POSTGRES_DB: manga_prod
    ports:
      - "5433:5432"
    volumes:
      - postgres_prod:/var/lib/postgresql/data

  api:
    image: manga-svc:prod
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://manga_prod:${POSTGRES_PASSWORD_PROD}@postgres:5432/manga_prod
```

Both tiers sit behind a single Cloudflare Tunnel — one `cloudflared` process, running as a launchd daemon on the Mac Mini — which means zero open inbound ports on the machine, no static IP requirement, and free Universal SSL for anything under the domain.

> Universal SSL only covers *one level* of subdomain under `*.genkos.app`, not two. That's why the routing is `genkos.app` (prod web), `api.genkos.app` (prod API), `staging.genkos.app` (staging web), and `staging-api.genkos.app` (staging API) — deliberately `staging-api`, not `api.staging`. Two levels of subdomain and the certificate simply doesn't cover it. `api.staging.genkos.app` is a trap, not a working form.

The staging web frontend sits behind Cloudflare Access, so it's reachable only by an authenticated allowlist; prod is public, because that's the point of prod. Same tunnel, same certificate coverage, different access policy per hostname — another property that's configured, not provisioned.

## Config, not code: how a live payment catalog moves between tiers

The part of this that surprised me most was billing. Wiring live Paddle price IDs into staging so testers can dry-run checkout without touching real payment infrastructure sounds like it should require a code branch — an `if environment == "staging"` somewhere in the checkout path. It doesn't.

The credit-pack catalog is read from a single environment variable, `PADDLE_PACKS`, expected to hold a JSON array of `{price_id, credits, display_price}` objects. When that variable is unset, the code falls back to a built-in sandbox catalog baked in as a constant — safe defaults that work against Paddle's sandbox mode out of the box:

```json
[
  { "price_id": "pri_live_xxxxxxxxxxxxxxxxxxxxxxxx", "credits": 5,  "display_price": "$4.99" },
  { "price_id": "pri_live_yyyyyyyyyyyyyyyyyyyyyyyy", "credits": 10, "display_price": "$8.99" },
  { "price_id": "pri_live_zzzzzzzzzzzzzzzzzzzzzzzz", "credits": 20, "display_price": "$15.99" }
]
```

Staging simply never sets `PADDLE_PACKS`, so it runs on the sandbox defaults. Prod sets it to the live-mode price IDs from the Paddle dashboard. It's the same binary, the same code path, reading a different value out of the environment at boot. Turning on real billing for a new tier isn't a code change or a deploy of new logic — it's setting one environment variable and restarting the container. That's the config-not-code property doing real work: the thing that used to feel like it needed a feature flag and a review is just an operations task.

## "But what happens when the one machine dies?"

This is the honest objection, and it deserves a real answer rather than a wave at "backups exist." A single Mac Mini is a single point of failure for compute — if it dies, both prod and staging go down at once, which two separate cloud accounts wouldn't guarantee. I didn't solve that; I made peace with a narrower promise: the *data* survives even if the hardware doesn't.

Every night at 03:00 UTC, a launchd job runs a `pg_dump` of both databases, gzips each one, and pushes it to a Cloudflare R2 bucket over the S3-compatible API — the same backup script, driven by which database names and which compose project it's pointed at, no environment-specific branching required. Roughly fourteen days of daily dumps are retained; anything older gets pruned on each run. Restoring is a documented one-liner: pull the dump back down from R2, pipe it through `gunzip`, pipe that into `psql` against a fresh Postgres instance. If the Mac Mini itself dies, I lose uptime and have to stand up a replacement — annoying, not fatal, and I know exactly how many hours of data I'm at risk of losing on any given day.

## Honest limits and where this stops being the right call

This setup has a ceiling, and it's worth naming rather than pretending it doesn't exist. One box has a real throughput limit — enough load and prod and staging start contending for the same CPU and disk. It has no meaningful answer for compliance regimes: a Mac Mini under a desk does not clear a SOC 2 audit or satisfy a data-residency requirement, no matter how clean the environment separation is internally. And it doesn't scale past one operator gracefully — the moment a second person needs deploy access, on-call rotation, or independent audit trails, the informal trust model this setup runs on stops working, and a managed platform's access controls start earning their cost rather than being overhead.

There's also a cost crossover, and it's worth being honest about which direction it points. The Mac Mini was roughly $599 one-time for a base M4, plus a Cloudflare Tunnel that's free at this scale — a sunk, one-time cost. The indicative recurring alternative — two small cloud instances, a managed Postgres tier, egress — runs into real monthly dollars that compound over a year in a way the Mac Mini never will, at least until the throughput ceiling above forces the issue. Cloud isn't selling isolation in that comparison — I already had that for free. It's selling *managed failover, audit trails, elastic capacity* — operational properties that are genuinely expensive to build yourself and genuinely worth paying for once you need them.

## The meta-lesson

The day-one instinct to open a cloud account for staging conflates two different purchases into one reflexive click. One is isolation — prod and staging not stepping on each other — which is a configuration property you get for free on hardware you already own, if you're disciplined about ports, volumes, database names, and environment variables. The other is a set of operational guarantees — geographic failover, compliance attestations, elastic scaling under load, someone else's on-call rotation — which is what cloud infrastructure actually sells, and which is worth real money the moment you need it.

The useful discipline isn't "avoid the cloud" or "always self-host." It's knowing which of the two you're buying at the moment you reach for your wallet. Graduate off the Mac Mini when you can name the specific operational property you're missing — not before, and not because opening an account felt like the professional thing to do on day one.
