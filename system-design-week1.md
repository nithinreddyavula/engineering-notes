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