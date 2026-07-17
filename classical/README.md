# Classical Systems

Traditional distributed systems design.

**Core concerns:** Scalability, Availability, Consistency (CAP), Latency, Throughput

## Cases

| System | Key Challenge |
|---|---|
| [URL Shortener](./cases/url-shortener/) | Read-heavy, hashing, redirects at scale |
| [Twitter Feed](./cases/twitter-feed/) | Fan-out, timeline generation, hot users |
| [Uber](./cases/uber/) | Geo-spatial indexing, real-time matching, surge pricing |

## Patterns

| Pattern | Used In |
|---|---|
| [Rate Limiting](./patterns/rate-limiting/) | API gateways, abuse prevention |
| [Caching](./patterns/caching/) | Feed, search, session |
| [Consistent Hashing](./patterns/consistent-hashing/) | Distributed caches, sharding |
