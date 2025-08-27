# ruebucket
ruebucket implements a redis-backed rate limiter using a **token bucket** algorithm.
Built on top of https://github.com/redis/rueidis, with auto-pipelining.

[Godoc](https://pkg.go.dev/github.com/iamcalledrob/ruebucket)

## Features
1. **Minimal resource usage:** When a bucket is full, no data is stored in Redis (default full design).
   Keys expire as soon as possible.
2. **Keyed limiters**: Supports applying limits per-identifier -- e.g. http requests by IP.
3. **Replenishable**: Tokens can be added back to the bucket on-demand.
4. **Local implementation**: A local (non redis) implementation of the same algorithm is provided for convenience.  
5. **Minimized round-trips to Redis**: Avoids constantly pinging Redis when a limiter is known to be exhausted.
   Handy for very hot paths.

## Goals
1. Be familiar to users of `golang.org/x/time/rate.Limiter`
2. Simple API, easy to use.
3. Be self-cleaning - no memory leaks, especially for keyed limiters.

## Usage
There are a few types available depending on requirements:

### KeyedLimiter
`KeyedLimiter` is a non-replenishable limiter that caches wait durations to avoid unnecessary redis calls
-- good for use on very hot paths where limiters will often be exhausted.

If an earlier call to Allow was unsuccessful due to an empty bucket, KeyedLimiter returns the wait time until
the next token will be available. The limiter caches this wait time, and won't make further round-trips to
redis until the wait time has elapsed.

This is the fastest limiter, and should be used unless replenishment is needed.

```go
// Allow 1 HTTP request per IP every second, with a burst of up to 100.
lim := ruebucket.NewKeyedLimiter(
    rueidisClient,
    "http_requests_by_ip",                // limiter identifier
    ruebucket.AllowEvery(time.Second),    // bucket replenishment rate
    100,                                  // bucket capacity (initially full)
)
```

```go
// Check if this IP is allowed to make another HTTP request
ok, wait, err := lim.Allow(context.Background(), "10.0.0.1")
if err != nil {
    // Failed to communicate with Redis
}
if !ok {
    // Not allowed -- no tokens left in the bucket
    // Next token available after 'wait'
    w.WriteHeader(http.StatusTooManyRequests)
    w.Header().Set("X-RateLimit-Remaining-Ms", strconv.FormatInt(wait.Milliseconds(), 10))
}

// Handle the request...
```

### Limiter
Limiter works just like `KeyedLimiter`, but limits a single default key.

```go
// Allow 1 signup every 5s, with a burst of up to 10.
signupsLimiter := ruebucket.NewLimiter(
    rueidisClient,
    "signups",                                // limiter identifier
    ruebucket.AllowEvery(5 * time.Second),    // bucket replenishment rate
    10,                                       // bucket capacity (initially full)
)
```

```go
ok, wait, err := signupsLimiter.Allow(context.Background())
if err != nil {
    // Failed to communicate with Redis
}
if !ok {
    // Not allowed -- no tokens left in the bucket
    // Next token available after 'wait'
}

// Allow signup
```

### ReplenishableKeyedLimiter
`ReplenishableKeyedLimiter` is a redis-backed token bucket rate limiter that supports attempting to take tokens
from the bucket AND replenishing tokens back into the bucket.

The limiter is identified by "id", and Allow/Replenish take a key used to identify an individual resource.
This allows a single limiter instance to limit multiple an action for multiple different actors,
e.g. limiting a common action (id: api_endpoint) on a per-actor basis (key: ip_address).

**Support for replenishing comes at the cost of increased Redis calls** -- since the number of tokens in a bucket can
change by more than just time, the client can't cache the wait until the next token will become available, and instead
has to ask Redis each time.

```go
// Manually replenish the bucket with a token
lim.Replenish(context.Background(), "10.0.0.1")
```

### ReplenishableLimiter
`ReplenishableLimiter` to `ReplenishableKeyedLimiter` is what `Limiter` is to `KeyedLimiter`.

### LocalKeyedLimiter / LocalLimiter
These limiters work in exactly the same way, using the same algorithm as their non-local Redis counterparts.
They exist for convenience. The only difference between the Local and Redis APIs is the lack of a `context` parameter.


## Benchmarks
Run with high parallelism in order to benchmark the limiter, not the RTT. Benchmarks run against Redis 8.0
running locally.

`BenchmarkLimiter/Limiter/Exhausted_Parallel` (322ns/op) highlights the performance gain of caching wait times
when the limiter is exhausted.

By contrast, `BenchmarkLimiter/Replenishable/Exhausted_Parallel100-8` is 20x slower as a replenishable limiter
is unable to cache waits.



```
goos: linux
goarch: amd64
pkg: github.com/iamcalledrob/ruebucket
cpu: 11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz
 BenchmarkLimiter
 BenchmarkLimiter/Limiter/Exhausted_Parallel100-8                 3903544              322.3 ns/op
 BenchmarkLimiter/Limiter/Allowed_Parallel100-8                    123291              9496 ns/op
 BenchmarkLimiter/Replenishable/Exhausted_Parallel100-8            178246              6485 ns/op
 BenchmarkLimiter/Replenishable/Allowed_Parallel100-8              117828              9901 ns/op
```
