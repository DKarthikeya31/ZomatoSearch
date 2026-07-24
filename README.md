# zomato-search-service

A small backend project where I built a restaurant/dish search service similar to how Zomato or Swiggy's search might work under the hood — focused on making it fast using concurrency and Redis caching, not just "make it work."

I built this because I wanted something on my resume that I could actually defend in an interview, not just a CRUD app. So every part of this — the concurrency, the caching, the retry logic — I built incrementally and timed the before/after myself.

`golang` `redis` `concurrency` `caching` `distributed-systems` `backend`

---

### Why this exists

When you search for a dish or restaurant on an app like Zomato, the backend usually has to:
- look up matching restaurants
- look up matching dishes
- combine them into one result

Doing this naively (one after the other) is slow. I wanted to actually build something that does this properly — in parallel, with caching so repeated searches don't hit the database again, and in a way that doesn't fall over if one part of the system is having a bad day.

### What it does

1. First checks Redis — if this exact search was made recently, return it instantly from cache.
2. If it's not cached, it fetches restaurant matches and dish matches **at the same time** (goroutines), instead of one after the other.
3. If one of those lookups fails temporarily, it retries a couple of times before giving up.
4. If one totally fails and the other succeeds, it still returns whatever it has instead of failing the whole request.
5. Once it has a result, it saves it to Redis in the background so the *next* search for the same thing is fast.

### Architecture (roughly)

```
Client
  │
  ▼
HTTP handler (main.go)
  │
  ▼
Search service
  │
  ├── check Redis cache — hit? return immediately
  │
  └── miss → run these two at the same time:
         ├── restaurant lookup (retries if it fails)
         └── dish lookup (retries if it fails)
              │
              combine → respond → save to Redis in background
```

Nothing fancy — the whole point was to actually build the fan-out + cache-aside pattern myself instead of just describing it in a system design interview.

### Stack

- **Go** — mainly because goroutines make the concurrency part clean to write
- **Redis** — for caching search results (`go-redis/v9` client)
- plain `net/http`, no framework — didn't need one for this

### How I actually built it (in case you want to build it yourself instead of copy-pasting)

I didn't write this all at once — I built it in stages so I could actually see each improvement working, rather than just writing "concurrency" on a resume and hoping nobody asks about it.

1. **Got the basic project running first** — just structs for Restaurant/Dish/SearchResult, and a main.go that starts and stops. Nothing else.
2. **Built the `/search` endpoint the dumb way first** — stub functions with `time.Sleep()` to fake real latency, called one after another. Timed it with curl so I had a "before" number.
3. **Made it concurrent** — wrapped both stub calls in goroutines with a WaitGroup. Timed it again — latency dropped from roughly the sum of both calls to roughly the max of the two. This is the number I actually put in my head for interviews.
4. **Added a timeout** so one slow call can't hang the whole request forever — tested this by making one stub sleep 2 seconds on purpose and watching it fail fast instead of hanging.
5. **Added Redis** — spun up Redis in Docker, wrote a small Get/Set wrapper, wired it in as check-cache-first. First call to a query is slow, second call is basically instant.
6. **Added retries** — made one of the stub functions fail on the first attempt (using a counter) just to prove the retry logic actually recovers from it.
7. **Handled partial failures** — made the dish lookup always fail and checked that restaurant results still came back instead of the whole thing erroring out.
8. **Cleaned it up** — added logging middleware, a `/healthz` endpoint, timeouts on the server itself, and jittered the cache TTL so a bunch of keys don't all expire at the exact same second.

If you're doing this yourself, I'd genuinely recommend going step by step like this instead of writing the whole thing in one sitting — you actually remember why each piece is there when you build it this way, which matters a lot more than the code itself when someone asks you about it later.

### Running it

```bash
docker run -p 6379:6379 redis:7-alpine
go mod tidy
go run main.go
```

```bash
curl "http://localhost:8080/search?q=biryani"
curl "http://localhost:8080/healthz"
```

First hit for a query is slower (goes to the "live" stub functions), anything after that within 5 minutes comes back from cache.

### What's real vs. fake here

Honestly, `searchRestaurants` and `searchDishes` are just stubs returning made-up data — I didn't wire this up to a real database or Elasticsearch, since the point was the concurrency/caching/retry logic around them, not building an actual search index. In a real job, those two functions would be swapped for real DB/search calls, but everything around them (the parallel fetch, the caching, the failure handling) would stay basically the same.

### Things I'd get asked about, and how I'd answer

**Why concurrency here specifically?**
Because the restaurant lookup and dish lookup don't depend on each other — there's no reason to wait for one before starting the other. Running them in parallel means the total time is roughly whichever one takes longer, not both added together.

**Why cache-aside and not something like write-through?**
Search results don't need to be perfectly fresh — a few minutes of staleness is fine for a search feature, so it's simpler to just check-then-fetch-then-cache rather than keeping the cache updated on every write.

**Why return partial results instead of failing?**
Because a broken dish index shouldn't mean the user gets nothing back at all. Some results is better than an error page.

**What does "production-grade" even mean in a small project like this?**
Timeouts everywhere so nothing hangs forever, retries that are capped (not infinite — that would just make things worse under load), actual logs you could debug with, and a health check endpoint that isn't just decorative.

---

One honest note: this is a personal project I built to *simulate* how Zomato/Swiggy-style search might work — I didn't actually work at either company. On my resume I'm listing it as a personal project, not real work experience, because that's the truth and it's an easy thing to get caught out on if asked directly.
