# Zomato-style Search Backend Service (Go + Redis)

A demo backend search service inspired by how large-scale food delivery
platforms (Zomato, Swiggy) handle restaurant/dish search: fast, concurrent,
cached, and resilient to partial failures.

This project backs the resume line:
> Developed scalable backend solutions for Zomato's search platform using
> Golang, Redis, and distributed systems. Optimized search performance
> through concurrency, caching strategies, and production-grade
> microservice enhancements.

## What it does

When a user searches for a restaurant or dish name, the service:
1. Checks Redis first (cache-aside pattern) — if the query was searched
   recently, the cached result is returned immediately.
2. On a cache miss, concurrently queries the restaurant lookup and the
   dish lookup at the same time (not one after another), cutting response
   time roughly in half versus a sequential approach.
3. Retries each lookup on transient failure with backoff, and tolerates
   partial failures — if one data source fails, results from the other
   are still returned instead of failing the whole request.
4. Writes the combined result back to Redis asynchronously with a
   jittered TTL, so repeat searches are fast without adding cache-stampede
   risk when many keys would otherwise expire at once.

## Architecture

```
Client
  │
  ▼
HTTP Handler (main.go)
  │  - logging middleware
  │  - timeout-bound context
  ▼
Search Service (internal/search)
  │
  ├──► Cache (Redis, cache-aside) ── cache hit? return immediately
  │
  └──► Cache miss: fan out concurrently
         ├── Restaurant lookup (with retry)
         └── Dish lookup (with retry)
              │
              ▼
         merge results → respond → async cache write
```

## Concepts demonstrated

| Concept | Where it shows up |
|---|---|
| **Concurrency** | `service.go` — restaurant and dish lookups run in parallel goroutines via `sync.WaitGroup`, instead of sequentially |
| **Caching** | `cache.go` — Redis cache-aside pattern, with jittered TTL to avoid stampedes |
| **Distributed systems thinking** | Context timeouts, partial-failure tolerance (one source can fail without failing the whole request) |
| **Scalability** | Connection pooling (`PoolSize: 50`), async cache writes so the hot path isn't blocked |
| **Production-grade enhancements** | Retry with backoff, structured request logging, `/healthz` readiness endpoint, write timeouts to avoid goroutine leaks from slow clients |

## Running it

```bash
# Requires Go 1.22+ and a local Redis instance
docker run -p 6379:6379 redis:7-alpine

go mod tidy
go run main.go
```

Then:
```bash
curl "http://localhost:8080/search?q=biryani"
curl "http://localhost:8080/healthz"
```

First call to a given query hits the live (simulated) data sources;
repeat calls within 5 minutes are served from Redis (`"source": "cache"`
in the response).

## What's stubbed vs. real

To keep this runnable as a standalone demo, `searchRestaurants` and
`searchDishes` in `service.go` return simulated data instead of querying
a real database or search index (Elasticsearch/Postgres). In a production
system, those two functions would be swapped for real client calls — the
concurrency, caching, retry, and failure-handling logic around them stays
the same.

## Build plan (how to build this yourself, incrementally)

Rather than starting from the finished code, build it in these batches —
each one leaves you with something that actually runs, so you're never
stuck with broken code for days, and you'll have a real story for *why*
each piece exists when it comes up in an interview.

### Batch 1: Project Setup & Data Models
- `go mod init zomato-search-service`
- Create `internal/models/models.go` — define `Restaurant`, `Dish`, `SearchResult`
- `main.go` that just prints "server starting" and exits
- **Checkpoint:** `go run main.go` compiles and runs

### Batch 2: Basic HTTP Server (sequential, no caching/concurrency yet)
- Set up `http.ServeMux`, `/search?q=...` route
- Stub `searchRestaurants()` / `searchDishes()` with `time.Sleep()` to simulate latency
- Call them **sequentially**, return combined JSON
- **Checkpoint:** `curl "localhost:8080/search?q=biryani"` works. Time it — note the latency (should be sum of both sleeps)

### Batch 3: Add Concurrency
- Introduce `sync.WaitGroup`
- Launch both stub calls in goroutines, store results in shared vars, `wg.Wait()` before merging
- **Checkpoint:** Re-time the same curl — latency should drop to roughly `max(a,b)` instead of `a+b`. **This before/after number is your resume proof point.**

### Batch 4: Add Context Timeouts
- Wrap the search call in `context.WithTimeout(ctx, 800*time.Millisecond)`, pass into both goroutines
- Test: make one stub sleep 2s, confirm the request fails/returns fast instead of hanging
- **Checkpoint:** You can explain what happens to a request when a dependency is too slow

### Batch 5: Redis Caching Layer
- `docker run -p 6379:6379 redis:7-alpine`
- Write `internal/cache/redis.go` with `Get`/`Set` using `go-redis`
- Check cache first → hit returns immediately; miss runs Batch 3's concurrent logic → writes result back
- **Checkpoint:** First call to a query is slow (`"source":"live"`), second call is fast (`"source":"cache"`)

### Batch 6: Retry Logic
- Generic `fetchWithRetry` helper: try, short backoff, retry once more (2 attempts total)
- Force a stub to fail on its first call (via a counter) to prove retry actually recovers it
- **Checkpoint:** Explain the difference between retrying and infinite-retry — why 2 attempts, not unlimited

### Batch 7: Partial Failure Handling
- After `wg.Wait()`, check both errors independently — only fail the whole request if **both** sources fail
- Force dish lookup to always error, confirm restaurant results still come back
- **Checkpoint:** Explain why "some result" beats "no result" for a search feature

### Batch 8: Production Hardening
- Structured request logging middleware (method, path, latency)
- `/healthz` endpoint that pings Redis
- `ReadTimeout`/`WriteTimeout` on `http.Server`
- Jittered TTL on cache writes (be ready to explain cache stampede in your own words)
- **Checkpoint:** Walk through every item here and say *why* it matters, not just that it's present

### Batch 9: Write Your Own README
Don't copy this one verbatim — write your own version covering:
- What the service does, in your own words
- The actual before/after latency numbers you measured in Batch 3
- One real bug you hit and fixed while building it (interviewers love this — it proves you actually built it, not copy-pasted it)

## Talking about this in an interview

Be ready to explain, in your own words:
- **Why concurrency helps here**: the restaurant and dish lookups are
  independent, so running them in parallel means total latency ≈
  `max(restaurant_time, dish_time)` instead of `sum(both)`.
- **Why cache-aside over write-through**: search queries are read-heavy
  and results tolerate a few minutes of staleness, so cache-aside with a
  short TTL is simpler and cheaper than keeping the cache in lockstep
  with every write.
- **Why partial failures return partial results**: a broken dish index
  shouldn't take down restaurant search entirely — availability of *some*
  useful result beats an all-or-nothing failure for a search feature.
- **What "production-grade" means here concretely**: timeouts everywhere
  (so one slow dependency can't hang the whole request), retries with
  backoff (not infinite retries), structured logs, and a health endpoint
  a load balancer can actually use.
