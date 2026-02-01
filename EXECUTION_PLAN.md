# Fraud System POC – Execution Plan

## Goal
Build a containerized fraud system POC locally (MacBook) and deploy the same containers to AWS to scale toward 5k TPS synchronous rule evaluation while avoiding:
- Data inconsistency
- Data loss
- Unmanageable operational complexity

## Non-Negotiable Design Principles
- **One source of truth:** PostgreSQL is authoritative.
- **Derived stores:** ClickHouse and OpenSearch are derived stores.
- **Synchronous rule decision:** The API request blocks on fraud decision.
- **Reliable async propagation:** Transactional outbox → Kafka → derived stores.
- **Language-agnostic rule engines:** Java and .NET execute the same rule definitions.

## High-Level Components

### Databases / Infrastructure (Containerized for POC)
- PostgreSQL (transactions, decisions, rules, outbox)
- Kafka (events backbone)
- ClickHouse (aggregation / feature store)
- OpenSearch (fuzzy match & watchlist)

### Application Services (Containers)
- Java Rule Engine (Spring Boot, Java 21) – primary
- .NET Rule Engine – parity / comparison
- Outbox Publisher (Postgres → Kafka)
- ClickHouse Consumer (Kafka → ClickHouse)
- OpenSearch Consumer (Kafka → OpenSearch)

## Phase 1 – Local Infrastructure (MacBook)
**Objective:** Stand up all infra reliably in Docker.

**Steps:**
1. Configure Docker Desktop (6 CPU, 12GB RAM).
2. Create `docker-compose.yml` with:
   - Postgres
   - Kafka + Zookeeper
   - ClickHouse
   - OpenSearch + Dashboards
3. Add health checks for all services.
4. Verify:
   - Postgres accepts connections
   - Kafka topics can be created
   - ClickHouse responds to queries
   - OpenSearch index creation works

**Deliverable:** Stable local infra running via `docker compose up`.

## Phase 2 – Data Model & Consistency Foundation
**Objective:** Guarantee no data loss and auditability.

**Steps:**
1. Create Postgres tables:
   - `transactions`
   - `decisions`
   - `rules`
   - `outbox_events`
2. Implement transactional outbox pattern:
   - transaction + decision + outbox written in **one DB transaction**
3. Define global IDs:
   - `txn_id`
   - `event_id`
   - `trace_id`

**Deliverable:** Reliable write model with replay capability.

## Phase 3 – Kafka Integration
**Objective:** Decouple write path from analytics/search.

**Steps:**
1. Create Kafka topics:
   - `transactions`
   - `decisions`
   - `dead-letter`
2. Implement Outbox Publisher:
   - poll `outbox_events`
   - publish to Kafka
   - mark published with retries + backoff
3. Ensure idempotent publishing.

**Deliverable:** Guaranteed event delivery to Kafka.

## Phase 4 – Derived Stores
**Objective:** Enable fast rules without hitting Postgres.

### ClickHouse
- Create append-only events table.
- Implement Kafka consumer:
  - batch inserts (not row-by-row)
- Support window queries:
  - count / sum / unique per key + time window

### OpenSearch
- Create watchlist index with analyzers.
- Implement Kafka consumer:
  - bulk indexing only
- Support fuzzy matching with score threshold.

**Deliverable:** Fast feature store + fuzzy search.

## Phase 5 – Rule Engines (SYNC)
**Shared Rule Model**
- Rules defined in JSON/YAML (not hardcoded)
- Same schema used by Java and .NET engines

### Java Rule Engine (Primary)
**Steps:**
1. Implement `POST /score`.
2. On request:
   - query ClickHouse features
   - query OpenSearch fuzzy/watchlist
   - apply last-mile correction (include current txn)
3. Evaluate rules.
4. Persist transaction + decision + outbox (one DB txn).
5. Return decision synchronously.

### .NET Rule Engine (Parity)
**Steps:**
- Mirror Java logic exactly.
- Same inputs, same rules, same outputs.
- Used for comparison / migration confidence.

**Deliverable:** Deterministic, synchronous fraud decision engine.

## Phase 6 – Testing & Proof
**Objective:** Prove scalability and correctness.

### Pipeline throughput test
- Generate 5k events/sec into Kafka.
- Verify ClickHouse & OpenSearch keep up (no lag growth).

### Sync scoring test
- Run sustainable sync TPS on MacBook.
- Capture p95 / p99 latency.

### Parity test
- Same input → Java vs .NET.
- Zero decision mismatch.

**Deliverable:** Metrics + evidence for design review.

## Phase 7 – AWS Deployment Plan (After POC)
**Objective:** Lift containers to AWS.

**Steps:**
1. Deploy rule engines & workers as containers on:
   - ECS (simpler) or EKS (more control)
2. Replace local infra with managed services:
   - Postgres → RDS
   - Kafka → MSK
   - OpenSearch → OpenSearch Service
   - ClickHouse → EC2/EKS cluster
3. Enable:
   - autoscaling
   - blue/green deployments
   - monitoring & alerts

**Result:** Same architecture, real 5k TPS capability.

## What This Plan Prevents
- ❌ data loss (outbox + replay)
- ❌ inconsistent decisions (single truth + last-mile correction)
- ❌ fragile deployments (stateless services, blue/green)
- ❌ single-language dependency (Java + .NET parity)

## Final Outcome
A defensible, scalable fraud platform:
- proven locally
- portable to AWS
- capable of synchronous high-TPS decisions
- auditable and operable
