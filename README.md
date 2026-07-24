<div align="center">

# 📦 ZomatoSearch

### Concurrent Search Backend for Restaurant & Dish Discovery

*Cutting search latency in half by fanning out restaurant and dish lookups in parallel — with Redis caching and graceful failure handling on top.*

![Go](https://img.shields.io/badge/Go-1.22-00ADD8?style=flat&logo=go&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-Cache--Aside-DC382D?style=flat&logo=redis&logoColor=white)
![Concurrency](https://img.shields.io/badge/Concurrency-Goroutines-00ADD8?style=flat)
![net/http](https://img.shields.io/badge/net%2Fhttp-API-informational?style=flat)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)
![Status](https://img.shields.io/badge/Status-Personal%20Project-yellow?style=flat)

[Problem](#-problem) · [Solution](#-solution) · [Architecture](#-architecture) · [Build Batches](#-build-batches) · [Getting Started](#-getting-started) · [Interview Notes](#-interview-notes)

</div>

---

## 🚀 Overview

A backend search service modeled on how apps like Zomato or Swiggy might
handle restaurant/dish search — built to actually be fast, not just
functional. Every optimization here (concurrency, caching, retries,
graceful degradation) was built incrementally and timed, so the resume
line behind it is backed by real numbers, not just buzzwords.

## 🧩 Problem

A naive search implementation looks up restaurants, then looks up dishes,
then combines them — sequentially. That means total response time is the
**sum** of both lookups, and a single slow or failing dependency takes the
whole request down with it. Neither is acceptable at any real scale.

## 💡 Solution

- **Fan out, don't queue up.** Restaurant and dish lookups run concurrently
  as goroutines, so total latency is roughly `max(a, b)` instead of `a + b`.
- **Cache what's already been asked.** A Redis cache-aside layer returns
  repeat queries instantly, with a jittered TTL so keys don't all expire in
  the same instant under load.
- **Expect things to fail, and don't collapse when they do.** Capped
  retries recover from transient blips; if one lookup still fails, the
  response returns whatever the other one found instead of erroring out
  entirely.

## 🏗️ Architecture

```
                     Client
                       │  GET /search?q=...
                       ▼
            ┌─────────────────────┐
            │   HTTP Handler        │
            │   (logging + timeout) │
            └──────────┬───────────┘
                       ▼
            ┌─────────────────────┐
            │    Search Service     │
            └──────────┬───────────┘
                       │
          ┌────────────┴─────────────┐
          ▼                          ▼
   Redis Cache hit?           Cache miss → fan out:
   → return instantly          ├─ Restaurant lookup (retry)
                               └─ Dish lookup (retry)
                                       │
                               merge → respond
                                       │
                        async write-back to Redis (jittered TTL)
```

## 🔖 Build Batches

Built incrementally, one working checkpoint at a time — not written in one
sitting, so every design choice below has an actual "before/after" behind it.

`BATCH-1` `SETUP` `MODELS` — project init, `Restaurant`/`Dish`/`SearchResult` structs, a `main.go` that starts and exits.

`BATCH-2` `HTTP` `SEQUENTIAL` — `/search` endpoint calling stubbed lookups one after another. Baseline latency measured here.

`BATCH-3` `CONCURRENCY` `GOROUTINES` — both lookups moved into goroutines under a `WaitGroup`. Latency drops from sum → max. **This is the number behind the resume line.**

`BATCH-4` `TIMEOUTS` `CONTEXT` — shared context deadline so one slow dependency can't hang the whole request.

`BATCH-5` `REDIS` `CACHE-ASIDE` — Redis wired in as check-cache-first; repeat queries return near-instantly.

`BATCH-6` `RETRY` `RESILIENCE` — capped retry with backoff on transient lookup failures.

`BATCH-7` `PARTIAL-FAILURE` `GRACEFUL-DEGRADATION` — one lookup failing no longer fails the whole request.

`BATCH-8` `LOGGING` `HEALTHCHECK` `HARDENING` — structured logging, `/healthz`, server-level read/write timeouts, jittered cache TTL.

## ▶️ Getting Started

```bash
docker run -p 6379:6379 redis:7-alpine
go mod tidy
go run main.go
```

```bash
curl "http://localhost:8080/search?q=biryani"
curl "http://localhost:8080/healthz"
```

First hit for a query goes to the (simulated) live path; anything within 5
minutes after that comes back from cache.

## 🔍 What's Stubbed vs. Real

`searchRestaurants` and `searchDishes` return simulated data — there's no
real database or search index behind this, since the point of the project
is the concurrency/caching/failure-handling layer around them, not the
search index itself. Those two functions are where real DB/Elasticsearch
calls would slot in without changing anything else.

## 🎯 Interview Notes

- **Why concurrency here:** the two lookups are independent, so there's no
  reason to serialize them — parallel execution drops latency to
  `max(a, b)`.
- **Why cache-aside over write-through:** search results tolerate a little
  staleness, so check-then-cache is simpler and cheaper than keeping the
  cache in lockstep with every write.
- **Why return partial results:** a broken dish index shouldn't mean the
  user gets nothing — some results beat an error page.
- **What "production-grade" means here:** timeouts everywhere, capped (not
  infinite) retries, structured logs, and a health check a load balancer
  can actually use.

---

*Personal project inspired by how food-delivery search platforms operate —
not a claim of employment at Zomato or Swiggy. Listed on my resume as a
personal project, not work experience.*
