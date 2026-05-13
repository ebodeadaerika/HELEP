# HELEP — Design Process Document (Part G)

---

## G.1 Problem Statement

Emergency response in Cameroon and similar markets faces three systemic failures: **slow dispatch** (phone-based coordination adds 5–15 minutes), **no visibility** (citizens have no feedback after an SOS call), and **poor analytics** (aggregated data for post-incident learning is unavailable).

HELEP solves these by providing a real-time, event-driven emergency dispatch platform where citizens submit SOS alerts via a mobile-friendly API, responders receive automated notifications, and operations teams monitor everything on a live dashboard.

---

## G.2 Stakeholders and Use Cases

| Stakeholder | Primary Need | Key Use Case |
|---|---|---|
| Citizen | Fast help, feedback | Submit SOS → receive status updates |
| Responder | Clear assignment | Receive dispatch notification with location |
| Dispatcher (ops) | Overview of active incidents | View all open SOS + assigned responders |
| Operations manager | Post-incident analytics | Weekly incident heatmap and response-time trends |
| Platform engineer | Reliable infra | Zero-downtime deployments, auto-scaling on surge |

---

## G.3 Architecture Decisions (ADRs)

### ADR-001 — Microservices over monolith

**Context:** The five functional domains (identity, SOS intake, dispatch logic, notification delivery, analytics) have different scaling profiles — SOS and dispatch spike during emergencies, analytics runs at low steady load.

**Decision:** Separate deployable service per domain, communicating via Kafka for async flows and direct HTTP for synchronous read paths (e.g. user-service token validation).

**Consequences:** (+) Independent scaling and deployment. (−) Operational overhead; mitigated by Helm umbrella chart + Strimzi.

### ADR-002 — Apache Kafka as event bus (not direct HTTP)

**Context:** The Saga `sos → dispatch → notification` must be resilient to partial failures. If notification-service is down when dispatch completes, the notification must still eventually be delivered.

**Decision:** All inter-service domain events flow through Kafka topics with consumer groups and offset management. HTTP is used only for synchronous queries.

**Consequences:** (+) Durability, replay, audit trail. (−) Eventual consistency; clients must poll or use WebSocket for status updates.

### ADR-003 — SQLite per service (dev/staging), Postgres in production

**Context:** The capstone uses SQLite with auto-migration for simplicity. Production requires concurrent writes and proper ACID guarantees.

**Decision:** `DB_PATH` env var controls the database. SQLite for dev/CI (no external dependency), Postgres for production (deployed as a StatefulSet or managed service).

**Consequences:** (+) Zero-dependency local dev. (−) DB migrations must be tested against both dialects.

### ADR-004 — Strategy pattern for responder matching

**Context:** Responder-matching rules vary by context (nearest, least-loaded, specialisation-matched). These must be swappable without redeployment.

**Decision:** `dispatch-service/app/matching.py` defines a `ResponderMatcher` protocol with concrete strategies injected via the `MATCHER` env var.

**Consequences:** (+) New matching strategies add a file, not a PR touching existing logic. (−) Requires factory/registry; included in `matching.py`.

### ADR-005 — Circuit breaker on all inter-service calls

**Context:** An unhealthy notification-service must not cascade to sos-service or dispatch-service.

**Decision:** Every downstream call is wrapped with `CircuitBreaker` from `dispatch-service/app/circuit_breaker.py`. State transitions (CLOSED → OPEN → HALF_OPEN → CLOSED) are logged via structlog and exposed on `/metrics`.

**Consequences:** (+) Bounded blast radius. (−) Callers must handle `CircuitBreakerOpenError` with a fallback (queue to retry, degrade gracefully).

---

## G.4 Data Flow — SOS Saga

```
Citizen
  │  POST /api/v1/sos  (JWT auth)
  ▼
sos-service
  │  Validates, persists incident (SQLite)
  │  Publishes → Kafka: sos-events
  ▼
dispatch-service (consumer group: dispatch-cg)
  │  Runs ResponderMatcher strategy
  │  Persists assignment
  │  Publishes → Kafka: dispatch-commands
  │  Publishes → Kafka: notification-events
  ▼
notification-service (consumer group: notification-cg)
  │  Sends SMS/push via gateway
  │  Publishes → Kafka: analytics-events
  ▼
analytics-service (consumer group: analytics-cg)
  │  Aggregates into time-series metrics
  └─ Exposes on /metrics (Prometheus)
```

Compensating transactions (saga rollback) are triggered by a `sos.dispatch_failed` event, which sos-service listens for and uses to mark the incident as unassigned and retry.

---

## G.5 Security Design

| Layer | Control |
|---|---|
| Transport | TLS on Kafka listener 9093; HTTPS enforced at Ingress |
| Authentication | JWT (HS256) signed with `JWT_SECRET`; validated in user-service and forwarded as `X-User-Id` header |
| Authorisation | Role field in JWT (`citizen`, `responder`, `admin`); enforced per endpoint |
| Network | Kubernetes NetworkPolicy: default-deny, explicit allow-list per service pair |
| Secrets | Kubernetes Secret `helep-secrets`; never in ConfigMap or image |
| Container | Non-root user (UID 1000), read-only root FS on all pods |
| Rate limiting | NGINX Ingress: 100 RPS per IP, 20 concurrent connections |

---

## G.6 Scalability and Reliability

- **HPA** on all services: CPU-triggered, min 2 replicas (no single point of failure).
- **Kafka partitions**: 6 for `sos-events` and `dispatch-commands` — supports 6 parallel consumer instances each.
- **PodDisruptionBudget** (recommended addition): `minAvailable: 1` per service to protect against voluntary disruptions during rolling upgrades.
- **Liveness / readiness probes** on all pods prevent traffic routing to unready instances.
- **Rollback**: `helm rollback helep <revision>` — tested in CI dry-run.

---

## G.7 Observability

- **Logs**: structlog JSON to stdout → collected by cluster log aggregator (Loki / CloudWatch).
- **Metrics**: Prometheus scrapes `/metrics` every 15s; Grafana dashboard `helep-main` shows SOS rate, dispatch latency p95, Kafka consumer lag, service error rate.
- **Alerts** (recommended): `KafkaConsumerLagHigh` (lag > 1000 for 5m), `ServiceErrorRateHigh` (5xx rate > 1% for 2m), `CircuitBreakerOpen` (any breaker in OPEN state).
- **Tracing**: OpenTelemetry SDK recommended as next iteration; trace IDs should be propagated in `X-Trace-Id` header.
