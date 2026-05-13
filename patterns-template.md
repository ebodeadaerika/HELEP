# HELEP — Patterns Document (Part H)

---

## H.1 Saga Pattern — Choreography-based Distributed Transaction

### What problem it solves

In a microservices system, a business transaction (citizen submits SOS → responder dispatched → citizen notified) spans multiple services. There is no distributed ACID transaction. If any step fails, we need a way to undo or compensate prior steps.

### How HELEP implements it

HELEP uses **choreography** (not orchestration): each service publishes events and reacts to events from others. There is no central coordinator.

```
Event flow (happy path):
sos-service        → publishes: sos.created         → topic: sos-events
dispatch-service   → publishes: dispatch.assigned   → topic: dispatch-commands
notification-svc   → publishes: notification.sent   → topic: notification-events

Compensating flow (dispatch fails):
dispatch-service   → publishes: sos.dispatch_failed → topic: sos-events
sos-service        → listens for sos.dispatch_failed
                   → marks incident status = UNASSIGNED, triggers retry
```

### Code location

- `services/sos-service/app/events.py` — publishes `sos.created`
- `services/dispatch-service/app/events.py` — consumes `sos-events`, publishes `dispatch.assigned` or `sos.dispatch_failed`
- `services/notification-service/app/events.py` — consumes `dispatch-commands`

### Trade-offs

| Pro | Con |
|---|---|
| No single point of failure | Harder to trace end-to-end flow |
| Services stay decoupled | Compensating logic in each service |
| Natural Kafka fit | Eventual consistency — not instant |

---

## H.2 Strategy Pattern — Responder Matching

### What problem it solves

Different emergency contexts require different matching logic: nearest responder in a city, least-loaded responder in a hospital network, specialisation-matched for hazmat. The algorithm must be swappable without changing the dispatch service's core logic.

### How HELEP implements it

A `ResponderMatcher` abstract base class defines the interface. Concrete strategies are registered by name. The `MATCHER` env var selects the active strategy at startup.

```python
# dispatch-service/app/matching.py (simplified)

from abc import ABC, abstractmethod

class ResponderMatcher(ABC):
    @abstractmethod
    def match(self, incident: Incident, responders: list[Responder]) -> Responder:
        ...

class NearestMatcher(ResponderMatcher):
    def match(self, incident, responders):
        return min(responders, key=lambda r: haversine(incident.location, r.location))

class LeastLoadedMatcher(ResponderMatcher):
    def match(self, incident, responders):
        return min(responders, key=lambda r: r.active_assignments)

class SpecialisationMatcher(ResponderMatcher):
    def match(self, incident, responders):
        qualified = [r for r in responders if incident.type in r.specialisations]
        return min(qualified or responders, key=lambda r: haversine(incident.location, r.location))

MATCHERS = {
    "nearest": NearestMatcher,
    "least_loaded": LeastLoadedMatcher,
    "specialisation": SpecialisationMatcher,
}

def get_matcher(name: str) -> ResponderMatcher:
    cls = MATCHERS.get(name, NearestMatcher)
    return cls()
```

### Adding a new strategy

1. Create a class that extends `ResponderMatcher` and implements `match()`.
2. Register it in `MATCHERS`.
3. Set `MATCHER=your_strategy` in the service's env vars.

No other code changes required.

---

## H.3 Circuit Breaker Pattern — Fault Isolation

### What problem it solves

If notification-service is slow or down, calls from dispatch-service will queue, consuming threads and memory, eventually cascading the failure upstream. A circuit breaker detects this and fast-fails subsequent calls until the dependency recovers.

### State machine (Part A.6 implementation)

```
        failure_count >= threshold
CLOSED ──────────────────────────────► OPEN
  ▲                                      │
  │                                      │ reset_timeout elapsed
  │                                      ▼
  │   success_count >= threshold     HALF_OPEN
  └────────────────────────────────────  │
                                         │ probe fails
                                         └───────────► OPEN (immediate)
```

| State | Behaviour |
|---|---|
| CLOSED | All calls pass through. Failure counter incremented on exception. |
| OPEN | All calls rejected with `CircuitBreakerOpenError`. Retry-After header returned to caller. |
| HALF_OPEN | One probe call allowed. Success → CLOSED. Failure → OPEN. |

### Code location

`services/dispatch-service/app/circuit_breaker.py`

### Usage example

```python
from app.circuit_breaker import dispatch_to_notification_cb

@dispatch_to_notification_cb.protect
async def send_notification(payload: dict) -> dict:
    return await http_client.post("http://notification-service:8004/notify", json=payload)

# Or imperatively:
try:
    result = await dispatch_to_notification_cb.call(send_notification, payload)
except CircuitBreakerOpenError as e:
    logger.warning("Notification circuit open, queuing for retry", retry_after=e.retry_after)
    await retry_queue.enqueue(payload, delay=e.retry_after)
```

### Metrics exposed

The `stats()` method returns `{name, state, failure_count, success_count}`. Prometheus gauge `helep_circuit_breaker_state` with label `name` is set to 0 (CLOSED), 1 (OPEN), or 2 (HALF_OPEN).

---

## H.4 Repository / Unit-of-Work Pattern — Data Access

### What problem it solves

Service handlers (`routes.py`) should not contain raw SQL. Swapping SQLite for Postgres in production must not require touching business logic.

### How HELEP implements it

Each service exposes a `Repository` class that abstracts all DB access. The `UnitOfWork` context manager handles transaction boundaries.

```python
# Pattern used in each service (example: sos-service)
class SosRepository:
    def __init__(self, conn: aiosqlite.Connection):
        self._conn = conn

    async def create(self, incident: SosIncident) -> SosIncident: ...
    async def get_by_id(self, incident_id: str) -> SosIncident | None: ...
    async def update_status(self, incident_id: str, status: str) -> None: ...

class UnitOfWork:
    async def __aenter__(self):
        self.conn = await aiosqlite.connect(settings.DB_PATH)
        self.sos = SosRepository(self.conn)
        return self

    async def __aexit__(self, exc_type, *_):
        if exc_type:
            await self.conn.rollback()
        else:
            await self.conn.commit()
        await self.conn.close()
```

---

## H.5 Anti-patterns Avoided

| Anti-pattern | Where it would have appeared | How HELEP avoids it |
|---|---|---|
| Distributed monolith | Services calling each other synchronously for every operation | Async Kafka events for domain flows; HTTP only for auth validation |
| God service | A single "emergency service" handling SOS, dispatch, and notifications | Strict single-responsibility per service |
| Chatty Kafka | Publishing an event for every database row change | Events represent domain facts (`sos.created`), not DB operations |
| Hardcoded secrets | JWT_SECRET in source | Kubernetes Secret + env var injection |
| Shared database | All services writing to one SQLite file | One DB per service; no cross-service joins |
