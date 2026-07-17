# Uber (Ride-Sharing Platform)

Design a ride-sharing system that matches riders to nearby drivers in real time, handles trip lifecycle, and computes dynamic pricing.

---

## Requirements

### Functional Requirements
- Rider requests a ride with pickup + destination
- System finds nearby available drivers
- Match rider to optimal driver (proximity, ETA, rating)
- Real-time GPS tracking of driver during trip
- Dynamic (surge) pricing based on supply/demand
- Trip lifecycle: request → match → pickup → in-trip → complete → payment
- Driver and rider rating after trip

### Non-Functional Requirements
- Latency: driver match < 500ms; location update propagation < 1s
- Availability: 99.99% — unavailability = lost revenue
- Consistency: eventual for location updates; strong for payment and trip state
- Security & Privacy: real-time location is sensitive PII; masked after trip
- Scalability: 10M trips/day, 5M concurrent drivers updating location every 4s

### Out of Scope
- Payment processing (integrate Stripe/Braintree)
- Driver onboarding / background checks
- Route navigation (use Google Maps API)
- Customer support tooling

---

## Back-of-Envelope Estimation

### Traffic Estimation
| Metric | Value |
|---|---|
| Trips/day | 10M |
| Peak concurrent trips | 1M |
| Active drivers (updating location) | 5M |
| Location updates/sec | 5M drivers ÷ 4s interval = 1.25M writes/sec |
| Ride requests/sec (peak) | ~2,000/sec |

### Storage Estimation
| Metric | Value |
|---|---|
| Trip record | ~1 KB |
| Trips/day | 10M → 10 GB/day |
| Trip history (5yr) | ~18 TB |
| Location update (ephemeral) | Not stored long-term; in-memory only |
| Location history for active trip | ~500 points → 8 KB/trip |

### Memory Estimation
| Metric | Value |
|---|---|
| Driver location index (5M drivers) | ~200 MB (lat/lng + metadata) |
| Geospatial index (in-memory) | ~2 GB with quad-tree overhead |
| Trip state cache (1M active trips) | ~1 GB |

### Bandwidth Estimation
| Metric | Value |
|---|---|
| Location update ingress | 1.25M × 50 bytes = ~62 MB/s |
| Driver location broadcast to riders | ~10 MB/s |

---

## High-Level Design

### Architecture Style
**Microservices + Event-driven** — location service decoupled from matching service; trip state changes published as events; each service owns its data store.

### Architecture Diagram

**Ride request flow:**
```
Rider App → API Gateway → Ride Request Service
                               ↓
                         Location Service (nearby drivers)
                               ↓
                         Matching Service (rank + select driver)
                               ↓
                         Notification Service (push to driver)
                               ↓ driver accepts
                         Trip Service (create trip, manage lifecycle)
                               ↓
                         Payment Service (on trip complete)
```

**Driver location update flow:**
```
Driver App (every 4s) → Location Service → Geospatial Index (in-memory)
                                                    ↓ pub/sub
                                          Rider App (during active trip)
```

**Real-time tracking (during trip):**
```
Driver App → Location Service → Kafka (location.updated)
                                         ↓
                               WebSocket Server → Rider App
```

### Core Components
| Component | Responsibility |
|---|---|
| Location Service | Ingest driver location updates; maintain geospatial index; query nearby drivers |
| Matching Service | Score and select best driver for ride request; handle driver accept/reject |
| Trip Service | Manage trip state machine; coordinate pickup → in-trip → complete |
| Surge Pricing Service | Compute dynamic multiplier from supply/demand ratio per geo-cell |
| Notification Service | Push notifications to driver (request) and rider (status updates) |
| Payment Service | Charge rider on trip completion; pay driver |
| WebSocket Service | Real-time bidirectional comms for location tracking |

### Data Flow

**Ride request:**
1. Rider opens app → requests ride with pickup (lat/lng) + destination
2. Surge Pricing Service returns multiplier for pickup area
3. Rider confirms → Ride Request Service creates request record
4. Location Service: geo-query for drivers within 5km radius, available status
5. Matching Service: score candidates by ETA + rating + acceptance rate → select top driver
6. Notification Service: push offer to driver (10s to accept)
7. Driver accepts → Trip Service creates trip; notifies rider with ETA
8. Driver rejects / no response → offer next candidate (retry up to 3×)

**During trip:**
1. Driver app sends GPS update every 4s → Location Service
2. Location Service publishes to Kafka topic `location.{trip_id}`
3. WebSocket Server consumes → pushes to rider's WebSocket connection
4. Trip Service monitors for trip completion (driver marks arrived at destination)
5. On complete: calculate fare (distance × rate × surge) → Payment Service charges rider

### Key Design Decisions
- **Geohash for spatial indexing:** Divide map into geohash cells (6-char geohash ≈ 1.2km²). Store drivers in `geohash → [driver_ids]` index. Query: find drivers in rider's cell + 8 adjacent cells. O(1) lookup per cell.
- **In-memory location index:** Driver locations change every 4s. DB writes at this rate (1.25M/sec) are too expensive. Keep in-memory (Redis sorted sets or custom quad-tree). Persist only completed trip routes.
- **Separate matching from location:** Location service is high-write (1.25M/sec updates); matching is low-write (2K/sec requests). Separate services scale independently.
- **WebSocket for real-time tracking:** Polling at 4s interval creates 250K requests/sec for 1M active trips. WebSocket maintains connection; server pushes updates — scales better, lower latency.
- **Trip state machine:** State transitions (request→matched→pickup→in-trip→complete→paid) persisted to DB with optimistic locking — prevents double-charging or double-matching.

---

## Low-Level Design

### Data Models
```
trips (PostgreSQL — strong consistency needed):
  trip_id         UUID      PK
  rider_id        UUID
  driver_id       UUID
  status          ENUM (requested, matched, pickup, in_trip, completed, cancelled)
  pickup_lat      FLOAT
  pickup_lng      FLOAT
  dest_lat        FLOAT
  dest_lng        FLOAT
  surge_multiplier FLOAT
  fare_cents      INTEGER   (set on completion)
  requested_at    TIMESTAMP
  matched_at      TIMESTAMP
  pickup_at       TIMESTAMP
  completed_at    TIMESTAMP
  version         INTEGER   (optimistic locking)

drivers (PostgreSQL):
  driver_id       UUID      PK
  name            TEXT
  rating          FLOAT
  vehicle_type    ENUM (economy, xl, black)
  acceptance_rate FLOAT     (rolling 30d)
  status          ENUM (offline, available, in_trip)

driver_locations (Redis — in-memory, ephemeral):
  Geohash index:
    key:   "geo:{geohash6}"
    type:  SET of driver_ids
    TTL:   30s (driver considered offline if no update in 30s)

  Driver position:
    key:   "loc:{driver_id}"
    type:  HASH { lat, lng, heading, updated_at, status }
    TTL:   30s

trip_route (S3 — archived after trip):
  trip_id/route.json → array of { lat, lng, timestamp }
```

### API Design
```
POST /rides/request
Request:  { "pickup": { "lat": 37.7, "lng": -122.4 }, "destination": { "lat": 37.8, "lng": -122.5 }, "vehicle_type": "economy" }
Response: { "ride_id": "...", "surge_multiplier": 1.8, "estimated_fare": "$12–15", "eta_pickup_min": 4 }

POST /rides/{ride_id}/confirm
Response: { "status": "searching", "ride_id": "..." }

GET /rides/{ride_id}/status
Response: { "status": "matched", "driver": { "name": "...", "rating": 4.9, "vehicle": "..." }, "eta_min": 3 }

WebSocket: /rides/{ride_id}/track
Server pushes: { "lat": 37.71, "lng": -122.41, "eta_min": 2, "heading": 270 }

POST /driver/location
Request:  { "lat": 37.71, "lng": -122.41, "heading": 270, "speed_kmh": 32 }
Response: 204

POST /driver/rides/{ride_id}/accept
Response: { "status": "accepted", "pickup": { "lat": ..., "lng": ... } }
```

### Component Deep-Dives

#### Geospatial Index — Geohash + Redis
```
Driver updates location:
  geohash6 = geohash.encode(lat, lng, precision=6)   # ~1.2km² cell
  geohash5 = geohash6[:5]                            # ~39km² parent

  pipeline = redis.pipeline()
  # Add to new geohash cell
  pipeline.sadd(f"geo:{geohash6}", driver_id)
  pipeline.expire(f"geo:{geohash6}", 60)
  # Update driver position
  pipeline.hset(f"loc:{driver_id}", mapping={"lat": lat, "lng": lng, "geohash": geohash6})
  pipeline.expire(f"loc:{driver_id}", 30)
  # Remove from old cell (if moved)
  if old_geohash != geohash6:
      pipeline.srem(f"geo:{old_geohash}", driver_id)
  pipeline.execute()

Find nearby drivers for rider at (lat, lng):
  rider_geohash = geohash.encode(lat, lng, 6)
  neighbors = geohash.neighbors(rider_geohash)  # 8 adjacent cells
  search_cells = [rider_geohash] + neighbors

  driver_ids = set()
  for cell in search_cells:
      driver_ids.update(redis.smembers(f"geo:{cell}"))

  # Filter: available only, get positions, compute actual distance
  candidates = []
  for did in driver_ids:
      loc = redis.hgetall(f"loc:{did}")
      if loc["status"] == "available":
          dist = haversine(lat, lng, loc["lat"], loc["lng"])
          if dist <= 5.0:   # km
              candidates.append((did, dist))
```

#### Matching Algorithm
```
Score each candidate driver:
  score = (
    w1 × normalize(ETA)          # lower ETA = higher score
  + w2 × normalize(driver_rating)
  + w3 × acceptance_rate
  + w4 × (1 if same vehicle_type else 0)
  )
  # weights: w1=0.5, w2=0.2, w3=0.2, w4=0.1

Select top-3 → offer to #1 (10s timeout)
  → rejected / timeout → offer #2
  → rejected / timeout → offer #3
  → still no match → expand radius to 10km, retry
```

#### Surge Pricing Service
```
Geohash5 cell (~39km² ≈ city neighborhood level)

Every 60s per active cell:
  supply = count(available_drivers in cell)
  demand = count(ride_requests in last 5min in cell)
  ratio  = demand / max(supply, 1)

  surge = 1.0                    # baseline
  if ratio > 2.0: surge = 1.5
  if ratio > 3.0: surge = 2.0
  if ratio > 5.0: surge = 3.0

  redis.set(f"surge:{geohash5}", surge, ex=120)

# Read at ride request time: instant, from Redis
```

#### Trip State Machine
```
States: REQUESTED → MATCHED → PICKUP → IN_TRIP → COMPLETED → CANCELLED

Transitions (with optimistic locking):
  UPDATE trips SET status='MATCHED', driver_id=?, version=version+1
  WHERE trip_id=? AND status='REQUESTED' AND version=?

  Rows affected = 0 → another process matched first → retry
```

#### Guardrails / Safety Layer
- **Driver identity verification:** real-time license plate check before trip starts
- **Anomaly detection:** GPS jumps > 200 km/h flagged as spoofed location
- **SOS button:** rider/driver can trigger emergency → shares live location with emergency contacts + Uber safety team
- **Location masking:** after trip, exact driver home location masked; only city-level retained
- **Surge cap:** max surge = 5× (regulatory in many markets); displayed prominently to rider

### Algorithms & Strategies
- **Geohash precision 6:** ~1.2km² cells → right granularity for city-level driver density. Precision 5 (~39km²) for surge pricing (neighborhood level).
- **Haversine distance:** exact great-circle distance for ranking candidates within geohash search result. O(N) on candidate set (usually < 50 drivers).
- **ETA computation:** call Google Maps / internal routing API for top-3 candidates after initial distance filter. Cache route ETAs per (driver_id, pickup) for 60s.
- **Driver location TTL (30s):** driver considered offline if no update in 30s. Automatic cleanup without explicit offline signal.
- **Payment fare calculation:** `fare = base_fare + (distance_km × per_km_rate) + (duration_min × per_min_rate)` × surge_multiplier. Calculated server-side on trip completion.

### Security Design
- AuthN: rider + driver authenticate via JWT; driver identity linked to verified government ID
- Location PII: driver real-time location shared only with matched rider; masked to approximate after trip
- Payment: card details stored by Stripe (PCI-DSS compliant); Uber stores only tokenized reference
- Fraud: detect fake GPS (speed anomaly check); detect fake trip completion (minimum distance check)
- Rate limiting: ride requests capped 10/min per rider; location updates 1/4s per driver
- Data residency: location data stored in region of operation (EU/US/IN separate)

---

## Observability

### Metrics
| Metric | Type | Alert threshold |
|---|---|---|
| Match latency (p99) | Histogram | > 500ms |
| Location update ingestion lag | Gauge | > 2s |
| Driver match rate | Gauge | < 85% (rides matched within 60s) |
| WebSocket connection drop rate | Counter | > 1% |
| Surge pricing stale | Gauge | > 2min since last compute |
| Trip state transition errors | Counter | > 0.01% (double-match prevention) |

### Logging
- Per ride request: ride_id, rider_id_hash, pickup_geohash, candidates_found, match_latency, driver_id_hash
- Per location update: driver_id_hash, geohash, latency, geohash_changed (bool)
- Per trip complete: trip_id, distance_km, duration_min, surge, fare_cents, rating

### Tracing
- Trace: ride_request → geo_query → matching → driver_notification → accept → trip_create
- Trace: location_update → geohash_index → kafka_publish → websocket_push
- Sample: 5% normal; 100% on match failures or latency > 1s

### Alerting
| Alert | Threshold | Action |
|---|---|---|
| Match rate drop | < 80% for 5min | Check location service, driver availability |
| Location lag | > 3s | Check Kafka consumer, location service load |
| Surge pricing stale | > 2min | Alert pricing team; fall back to last known |
| Payment failure | > 0.1% | Page payment team |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| In-memory geospatial index (Redis) | 1.25M location updates/sec | TTL-based eviction; lose location on Redis restart |
| Geohash over quad-tree | Simple, Redis-native, fast | Less precise boundary handling at cell edges |
| WebSocket over polling | Efficient real-time tracking | Connection management complexity |
| Separate location + matching services | Independent scaling | Cross-service calls on match path |
| Surge at geohash5 (~39km²) | Neighborhood-level accuracy | Too coarse for small event-driven surges |
| PostgreSQL for trip state | ACID for billing correctness | Can't scale writes horizontally beyond sharding |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Location index cold (Redis restart) | OOM or node failure | Drivers repopulate within 4s naturally; brief match degradation |
| Double match | Race on driver accept | Optimistic locking on trip state transition |
| Driver GPS spoof | Fake location to game surge | Speed anomaly detection; suspend driver account |
| Surge pricing stale | Compute job lag | TTL 120s; serve last known; alert if > 2min |
| WebSocket disconnect | Network instability | Client reconnects; buffers missed location updates |
| Payment failure | Card declined or gateway down | Retry with backoff; suspend account after 3 failures |

---

## Feedback Loop & Improvements
- Driver ETA model: improve ETA accuracy using historical trip data per route/time-of-day
- Surge price elasticity: measure demand drop vs surge level → optimize multiplier for revenue + supply
- Match quality: track rider rating of driver after match → feedback into matching weights
- Predicted demand: ML model predicts surge 15min ahead → notify drivers to move to high-demand areas

---

## Interview Tips

### What Interviewers Expect
- Geohash for spatial indexing — explain cell size, neighbor search, Redis storage
- In-memory driver location (not DB) — justify with write volume math (1.25M/sec)
- Trip state machine with optimistic locking — prevent double match/charge
- Fan-out problem: 1 location update → 1 rider WebSocket push (not same as Twitter fan-out, but same concept)

### Common Follow-ups
- How to handle surge pricing? Supply/demand ratio per geohash5 cell, computed every 60s
- How to scale location service to global? Shard by region; each city's drivers in regional Redis cluster
- Driver goes offline mid-trip? Trip continues; last known location cached; reconnect on next ping
- How to detect GPS spoofing? Speed > 250 km/h between consecutive pings = anomaly → flag + review
