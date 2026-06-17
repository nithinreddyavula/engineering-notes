# System Design Notes — Week 1

## Universal Framework
1. Requirements — functional and non-functional
2. Capacity estimation — DAU, QPS, storage
3. High Level Design — components and connections
4. Deep Dive — critical components in detail
5. Bottlenecks — identify and resolve

## Load Balancer
- Sits between client and server pool
- Solves two problems: uneven load distribution and single point of failure
- If one server goes down, load balancer routes to remaining healthy instances

### Round Robin
- Distributes requests in fixed circular sequence
- Simple, works well when all requests have similar processing time
- Weakness: does not account for server load

### Least Connections
- Routes each new request to server with fewest active connections
- Better for BookMyShow where requests have variable processing time
- Cache hit = 5ms, cache miss + MySQL write = 200ms
- Least Connections prevents overloading slow servers

## Cache Aside Pattern
- Application checks cache first before querying database
- Cache hit: return cached value immediately
- Cache miss: query database, populate cache, return value
- TTL prevents stale data
- BookMyShow URL Shortener: key = short code, value = original URL, TTL = 24 hours

## Redis SET NX
- SET key value NX EX ttl
- NX: only set if key does not exist — atomic operation
- EX: TTL in seconds — prevents deadlock if payment abandoned
- BookMyShow: SET seat:lock:{showId}:{seatId} {userId} NX EX 300
- First user gets OK, second user gets null — rejected with 409
- Chosen over MySQL row locking because Redis handles 100k ops/sec vs
  MySQL connection pool exhaustion under concurrent load
  # Rate Limiter Design

## Why It Exists
Restricts how many requests a single user/IP/key can make in a time window, to protect backend from abuse. Does NOT solve general traffic scaling (load balancer's job) or downstream outages (e.g. MySQL down).

## Fixed Window Counter
Counts requests per fixed time block (e.g. 0-60s). Simple but has a boundary exploit: 100 requests at 11:59:58 + 100 at 12:00:01 = 200 requests in 3 seconds, but both windows independently show 100 and allow it.

## Sliding Window Log
Stores every request timestamp, counts those within the trailing window. Accurate but memory-expensive at BookMyShow's scale (millions of users).

## Sliding Window Counter (production choice)
Weighted hybrid of previous and current window counts:
Example: 30% into current window → `100 × 0.70 + 20 = 90`. Fixes the boundary exploit with low memory overhead.

## Token Bucket
Bucket holds N tokens, refills at a fixed rate, each request consumes one token. Allows natural bursts (good for real user click patterns) but not what I use for the boundary-exploit fix — sliding window counter is more memory-efficient at scale.

## Distributed Counter Problem
With multiple Spring Boot instances behind a load balancer, local in-memory counters can't see each other. User sends 90 requests to Instance 1 and 90 to Instance 2 — both allow it locally, but 180 requests actually hit the backend against a 100 limit.

**Fix:** Move the counter to Redis, shared across all instances.
