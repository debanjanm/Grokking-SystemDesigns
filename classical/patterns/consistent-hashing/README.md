# Pattern: Consistent Hashing

Distribute keys across nodes such that adding or removing a node remaps only ~1/N keys — not all keys.

---

## What It Is

Standard modulo hashing (`key % N`) breaks when N changes — every key remaps to a different node, causing a cache miss storm or data migration avalanche. Consistent hashing maps both keys and nodes onto a hash ring; a key routes to the nearest node clockwise. Adding/removing a node only affects its immediate neighbors.

---

## When to Use

| Use | Avoid |
|---|---|
| Distributed cache (Redis Cluster, Memcached) | Fixed node count that never changes |
| Sharded databases (horizontal partitioning) | Simple single-node systems |
| Distributed message queues (partition assignment) | When rebalancing cost is irrelevant |
| Load balancing with session affinity | Stateless services (use round-robin instead) |
| Content delivery / CDN routing | |

---

## Core Concepts

### The Problem with Modulo Hashing
```
3 nodes: key → node = hash(key) % 3

Add 1 node (now 4 nodes):
  hash(key) % 3  ≠  hash(key) % 4  for most keys
  → ~75% of keys remap to different node
  → cache miss storm; data migration required
```

### Hash Ring
```
Ring: hash space [0, 2^32) arranged as a circle

Nodes placed on ring:
  hash("node-A") = 10  → position 10
  hash("node-B") = 50  → position 50
  hash("node-C") = 80  → position 80

Key routing: hash(key) → walk clockwise → first node encountered
  hash("key1") = 30  → routes to node-B (next clockwise from 30)
  hash("key2") = 60  → routes to node-C
  hash("key3") = 95  → routes to node-A (wraps around)
```

### Adding/Removing a Node
```
Add node-D at position 45:
  Before: keys 10–50 → node-B
  After:  keys 10–45 → node-B, keys 45–50 → node-D

Only keys in [45, 50] remap — ~1/4 of node-B's keys.
All other nodes unaffected.

Remove node-B:
  Keys formerly going to node-B → now go to node-D (next clockwise)
  Only node-B's keys remap.
```

### Virtual Nodes (vnodes)
Problem: with few real nodes, ring is uneven → hotspots.

```
Real nodes → each mapped to K virtual positions on ring
  node-A → positions: 12, 47, 83, 156, ...   (K=150 vnodes each)
  node-B → positions: 5, 31, 72, 201, ...
  node-C → positions: 23, 68, 99, 180, ...

Result: keys distribute evenly across nodes even with few real nodes
Rule of thumb: K = 100–200 vnodes per node
```

---

## Architecture

### Data Structure — Sorted Ring
```
Internally: sorted array (or sorted map) of (hash_position, node_id)

ring = SortedMap:
  5   → node-B-vnode-1
  12  → node-A-vnode-1
  23  → node-C-vnode-1
  31  → node-B-vnode-2
  47  → node-A-vnode-2
  ...

Lookup: binary search for first position >= hash(key) → O(log N)
```

### Replication
```
For fault tolerance, replicate each key to R nodes:
  hash(key) = 30 → position 30
  Replica 1: first node clockwise  = node-B (position 50)
  Replica 2: second node clockwise = node-C (position 80)
  Replica 3: third node clockwise  = node-A (wraps)

Read/write quorum (W + R > N for strong consistency):
  N=3, W=2, R=2 → quorum reads/writes
```

### Node Failure Handling
```
node-B goes down:
  Reads: go to next replica (node-C) — if N=3, R=2, one failure tolerated
  Writes: hinted handoff — write to node-C with hint "belongs to node-B"
  Recovery: when node-B returns, replay hinted writes (anti-entropy)
```

---

## Implementation Details

### Python — Consistent Hash Ring
```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, nodes=None, vnodes=150):
        self.vnodes = vnodes
        self.ring = {}          # position → node
        self.sorted_keys = []   # sorted positions

        for node in (nodes or []):
            self.add_node(node)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        for i in range(self.vnodes):
            vkey = f"{node}:vnode:{i}"
            pos = self._hash(vkey)
            self.ring[pos] = node
            bisect.insort(self.sorted_keys, pos)

    def remove_node(self, node: str):
        for i in range(self.vnodes):
            vkey = f"{node}:vnode:{i}"
            pos = self._hash(vkey)
            del self.ring[pos]
            self.sorted_keys.remove(pos)

    def get_node(self, key: str) -> str:
        if not self.ring:
            return None
        pos = self._hash(key)
        idx = bisect.bisect(self.sorted_keys, pos) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

    def get_nodes(self, key: str, n: int) -> list:
        """Get n distinct nodes for replication"""
        if not self.ring:
            return []
        pos = self._hash(key)
        idx = bisect.bisect(self.sorted_keys, pos) % len(self.sorted_keys)
        seen = set()
        result = []
        for i in range(len(self.sorted_keys)):
            node = self.ring[self.sorted_keys[(idx + i) % len(self.sorted_keys)]]
            if node not in seen:
                seen.add(node)
                result.append(node)
            if len(result) == n:
                break
        return result
```

### Key Remapping on Node Change
```
Before (3 nodes, vnodes=150): 450 positions on ring
After adding node-D:           600 positions on ring

Keys remapped = keys previously in segments now claimed by node-D
Expected: ~1/4 of total keys (N+1 nodes → 1/(N+1) remapped)
Actual variance reduced by vnodes → close to theoretical 1/N
```

### Hotspot Mitigation
```
Problem: certain keys much more popular (celebrity user, viral content)
Consistent hashing doesn't solve hotspots — it distributes keys evenly, not load

Solutions:
  1. Add read replicas for hot nodes
  2. Client-side local cache (absorb hot reads before ring lookup)
  3. Key sharding: "celebrity_user_id:{shard_0..9}" → 10× spread
  4. Detect hot keys via monitoring → special-case handling
```

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| Consistent hashing over modulo | ~1/N remapping on change | Complexity vs simple `key % N` |
| Virtual nodes (vnodes) | Even distribution | More memory for ring data structure |
| More vnodes (150+) | Better load balance | Slower node add/remove (more ring updates) |
| Replication factor N=3 | Fault tolerance | Storage cost 3× |
| W=2, R=2 quorum | Strong consistency | Higher latency (wait for 2 acks) |
| W=1, R=1 | Low latency | Eventual consistency; stale reads possible |

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Too few vnodes (< 50) | Uneven distribution; use 100–200 |
| Not handling wraparound on ring | Modulo `len(sorted_keys)` on index |
| Hotspot ignored | Monitor node load; add replicas for hot nodes |
| Node identity not stable | Node IDs must be deterministic across restarts (use hostname, not random) |
| No replication | Single node failure loses all its keys |
| Thundering herd on node removal | Gradual traffic drain before removal; replica takes over smoothly |

---

## Real-World Usage

| System | How they use it |
|---|---|
| **Amazon DynamoDB** | Consistent hashing for partition routing |
| **Apache Cassandra** | Token ring; vnodes for data distribution |
| **Redis Cluster** | 16384 hash slots mapped to nodes (variant of consistent hashing) |
| **Memcached** (client-side) | Client-side consistent hashing for cache routing |
| **Chord DHT** | Foundational P2P DHT algorithm based on consistent hashing |
| **Nginx upstream** | `consistent_hash` directive for session-affinity load balancing |

---

## Used In
- [URL Shortener](../../cases/url-shortener/) — Redis Cluster uses consistent hashing for cache sharding
- [Twitter Feed](../../cases/twitter-feed/) — Cassandra uses token ring for tweet/timeline data sharding
- [Uber](../../cases/uber/) — distributed session store sharding; geo-partition routing
- [Recommendation Engine](../../../ml-systems/cases/recommendation-engine/) — feature store Redis Cluster sharding

---

## Variants

| Variant | Notes |
|---|---|
| **Rendezvous hashing** (HRW) | Each key scores all nodes; pick highest score. No ring needed. Better balance; O(N) lookup |
| **Jump consistent hash** | Deterministic; O(log N) lookup; can't remove arbitrary nodes |
| **Maglev hashing** | Google's variant; optimized for load balancer consistency at scale |
| **Redis hash slots** | 16384 fixed slots; nodes own slot ranges; simpler than pure ring |
