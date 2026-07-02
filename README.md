# Rydex

Real-time ride-hailing and delivery dispatch platform.

**Status:** In development — architecture, domain model, and API contract are finalized; implementation is proceeding against the phased roadmap below.

> "Rydex" is a working title. The architecture, scope, and engineering rigor documented here are what matter — the name may change without affecting any of the substance below.

---

## Table of Contents

1. [Overview](#overview)
2. [Scope](#scope)
3. [Personas](#personas)
4. [Architecture](#architecture)
   - [Service Topology](#service-topology)
   - [Technology Stack](#technology-stack)
   - [Domain Model](#domain-model)
   - [Bounded Contexts](#bounded-contexts)
   - [Data Architecture](#data-architecture)
   - [Real-Time Communication](#real-time-communication)
   - [Messaging and Event Backbone](#messaging-and-event-backbone)
   - [Matching Engine](#matching-engine)
   - [External Integrations](#external-integrations)
5. [Architecture Decision Records](#architecture-decision-records)
6. [API](#api)
7. [Repository Structure](#repository-structure)
8. [Getting Started](#getting-started)
9. [Testing](#testing)
10. [Observability](#observability)
11. [Roadmap](#roadmap)
12. [Success Metrics](#success-metrics)
13. [Service Level Objectives](#service-level-objectives)
14. [Risk Register](#risk-register)
15. [Documentation](#documentation)
16. [Contributing](#contributing)
17. [License](#license)

---

## Overview

Rydex connects riders and senders with nearby drivers, matches them intelligently, prices trips dynamically based on live supply and demand, and keeps both sides of the marketplace informed throughout a trip in real time.

The project is built to demonstrate that a small, disciplined engineering team can design and ship a system with the architectural rigor of a production ride-hailing platform — load-tested, monitored, and reasoned about the way real production systems are — within a deliberately bounded scope.

**Core problem addressed:** connecting someone who needs a ride or delivery with someone available to provide it — fast, fairly, and safely — at a moment's notice, at any volume of demand. This is difficult because supply is constantly moving, demand is spiky, trust between strangers transacting through an app is not optional, and any latency in the experience is felt immediately by the user.

**Vision:** a dispatch platform where every match is fast and fair, every price reflects real conditions, every trip is trackable in real time, and the system itself is observable enough that problems are caught before users notice them.

**Strategic thesis:** Rydex does not compete on feature breadth. It competes on depth of correctness in a narrow scope — the same categories of decision a production system requires (consistency tradeoffs, service boundaries, load evidence) made with the same seriousness, in a single-region demonstration system rather than a multi-region commercial product.

## Scope

**Rydex is:**
- A single-region, real-time marketplace connecting riders/senders with drivers
- Intelligent matching (nearest-driver at launch, batch and fairness-aware matching in later milestones)
- Dynamic, zone-based surge pricing that responds to measured supply/demand imbalance
- Live, two-sided trip tracking over persistent connections
- A trust and safety layer with human-in-the-loop fraud review
- A retrieval-grounded AI support assistant with honest escalation

**Rydex deliberately is not:**
- A multi-city commercial operation — it is architected to behave like one, but launches as a single-region demonstration system
- A payments processor — it integrates with a real payment provider rather than building one
- A mapping or routing provider — it integrates with existing map and routing data rather than building proprietary maps

This boundary is stated explicitly and repeatedly across the project's documentation so that scope is never mistaken for incompleteness.

## Personas

| Persona | Core need |
|---|---|
| **Rider** | Fast matching, accurate ETA, transparent pricing, a sense of safety |
| **Driver** | Fair access to ride offers, predictable earnings, low-friction navigation |
| **Sender** (delivery variant of Rider) | Reliable pickup/drop-off, live tracking, proof of delivery |
| **Ops / Support Agent** | Visibility into trips, ability to intervene, fast dispute resolution tools |
| **Platform Admin** | Control over pricing rules, zones, and driver onboarding without needing engineering support |

## Architecture

### Service Topology

Rydex is a **modular monolith through Alpha and Beta 1**. All bounded contexts — Identity, Trip Lifecycle, Matching, Pricing, Trust & Safety, AI Support — live in a single deployable ASP.NET Core application, with internal module boundaries enforced by Clean Architecture layering and vertical-slice organization.

**Starting Beta 2**, the Matching Engine and Fraud Service are extracted into separate processes. This timing is deliberate: Beta 2 is the first point at which Matching's CPU-bound, latency-critical workload genuinely diverges from the rest of the system under realistic load, and Beta 3 is when the Fraud Service's asynchronous, off-critical-path nature becomes load-bearing. The extraction points are designed into the solution structure from day one so the later split is a code move, not a redesign.

```
Alpha / Beta 1                         Beta 2+
┌─────────────────────────┐            ┌──────────────────┐  ┌──────────────┐
│   Core Application       │            │ Core Application  │  │  Matching    │
│  (Trip, Auth, Pricing,   │  ───────►  │ (Trip, Auth,      │  │  Engine      │
│   Matching, Fraud —      │            │  Pricing)         │  │  (separate)  │
│   internal modules)      │            └──────────────────┘  └──────────────┘
└─────────────────────────┘                                   ┌──────────────┐
                                                                │ Fraud        │
                                                                │ Service      │
                                                                │ (separate)   │
                                                                └──────────────┘
```

### Technology Stack

| Concern | Technology | Decision Record |
|---|---|---|
| API and application runtime | ASP.NET Core (REST + JSON) | ADR-005 |
| Transactional data store | PostgreSQL — EF Core for writes, Dapper for read-heavy/paginated queries | ADR-002 |
| Live location and cache | Redis (`GEOADD`/`GEOSEARCH`, fairness queue, SignalR backplane) | ADR-003 |
| Message broker | RabbitMQ + MassTransit | ADR-004 |
| Real-time transport | SignalR over a Redis backplane | — |
| Background processing | .NET `BackgroundService` (Outbox relay) | ADR-006 |
| Vector store (RAG) | pgvector (PostgreSQL extension) | ADR-007 |
| Service topology | Modular monolith → extracted Matching Engine / Fraud Service at Beta 2/3 | ADR-001 |
| Internal service protocol | REST + JSON (gRPC is a stretch-tier optimization, not adopted preemptively) | ADR-005 |
| Observability | Serilog, OpenTelemetry, Prometheus, Grafana, Jaeger | — |
| CI/CD | GitHub Actions | — |
| Containerization | Docker / Docker Compose | — |

### Domain Model

Rydex's domain model defines exactly **three aggregates**:

| Aggregate | Root | Responsibility |
|---|---|---|
| **Trip** | `Trip` | Owns the trip state machine and is the only object through which a status transition may occur |
| **Driver** | `Driver` | Owns identity-level invariants — onboarding status, suspension status, document verification — independent of presence |
| **Rider** | `Rider` | Owns identity-level invariants for the rider side — account status, saved payment method reference, saved locations |

**Trip state machine:**

```
Requested → Matched → DriverEnRoute → InProgress → Completed | Cancelled
```

- `Requested` — a Trip has been created; no Driver is yet assigned.
- `Matched` — a candidate Driver has been offered the Trip. This does **not** mean the Driver has accepted; it means an offer is outstanding.
- `DriverEnRoute` — the offered Driver has accepted and is traveling to pickup. This is the first state that confirms Driver commitment.
- `InProgress` — the Rider has been picked up; the trip is underway.
- `Completed` / `Cancelled` — terminal states. No further transitions are permitted from either.

Legal transitions are enforced in **both** the domain layer (the `Trip` aggregate's own methods) and a database constraint, so the invariant holds regardless of entry point.

**Value objects:** `TripId`, `DriverId`, `RiderId`, `Coordinate` (immutable WGS84 lat/lng pair), `FareAmount`.

**Domain events:** `TripRequestedEvent`, `TripMatchedEvent`, `TripStatusChangedEvent`, `TripCompletedEvent`, `DriverProfileUpdatedEvent`, `RiderProfileUpdatedEvent`.

### Bounded Contexts

| Context | Owns | Primary aggregate(s) |
|---|---|---|
| **Identity** | Rider/Driver identity facts: account existence, role, authentication, onboarding and suspension status | Driver, Rider (identity fields) |
| **Trip Lifecycle** | The Trip aggregate, its state machine, and the append-only TripEvent history | Trip |
| **Matching** | Finding and offering a Driver to a Trip; the fairness queue; Driver presence | None (reads Trip and Driver by reference) |
| **Pricing** | Fare calculation logic, surge multiplier, Zone definitions | None (embeds `FareAmount` into Trip via Trip Lifecycle) |
| **Trust & Safety** | Fraud/anomaly scoring, disputes, SOS flags | None (reads TripEvent history; writes flags by reference) |
| **AI Support** | Retrieval, conversation context, escalation state | None (references Rider/Driver by ID) |

Driver **presence** (online/offline/busy) is owned by the Matching context, not Identity — presence is Redis-backed, TTL-expiring, and consumed almost exclusively by dispatch, while Identity remains the durable source of truth for the Driver record itself.

### Data Architecture

Rydex uses three logically distinct stores, each chosen for the correctness/performance tradeoff its data actually needs:

| Store | Technology | Holds |
|---|---|---|
| Transactional store | PostgreSQL | Trips, Drivers, Riders, Fares, Payments |
| Live-state store | Redis | Driver location, fairness queue, SignalR backplane |
| Event/history store | PostgreSQL (append-only) | `TripEvent` log — audit trail and fraud-signal source |

**Concurrency:** optimistic concurrency via a `RowVersion` column on `Trip` and `Driver`, used specifically to prevent the double-booking race — two near-simultaneous match attempts assigning the same Driver to different Trips. A losing update returns `409 Conflict` rather than silently overwriting state.

**Idempotency:** `Trip.IdempotencyKey` carries a unique constraint so a retried `POST /trips` request can never create a duplicate Trip.

**Location data availability:** Redis is the primary system of record for live driver location, chosen for its sub-millisecond `GEOSEARCH` performance. If Redis becomes unavailable, matching falls back to a PostGIS/SQL geography query — slower, but available, consistent with a graceful-degradation-over-full-outage design.

### Real-Time Communication

Live location and trip status updates are delivered over a SignalR hub backed by a Redis backplane, so message delivery is correct across more than one backend instance. Two reliability tiers are modeled explicitly and never conflated:

| Channel | Reliability tier | Rationale |
|---|---|---|
| Location pings | Lossy-acceptable | A dropped ping just means a slightly stale position |
| Trip status changes | Must-not-lose | A dropped status event (e.g., "trip completed") is a correctness failure, not just a latency miss |

### Messaging and Event Backbone

Domain events are written to an `OutboxMessages` table in the same transaction as the state change they represent (the transactional Outbox pattern), guaranteeing an event is never lost even if the broker is briefly unavailable. A polling `BackgroundService` relays unprocessed rows to RabbitMQ via MassTransit, checking `ProcessedMessages` for idempotency before each publish — consumers are required to tolerate at-least-once delivery.

### Matching Engine

Matching is implemented behind a single, stable interface so new strategies can be added without touching the dispatch service:

```csharp
public interface IMatchingStrategy
{
    Task<MatchResult> FindMatch(MatchRequest request, IReadOnlyList<AvailableDriver> candidates);
}
```

| Strategy | Ships | Approach |
|---|---|---|
| `NearestDriverStrategy` | Alpha | Haversine distance to each candidate; selects the minimum |
| `BatchMatchingStrategy` | Beta 2 | Hungarian algorithm over the full requests × candidates cost matrix; minimizes aggregate pickup distance across simultaneous requests |
| `FairnessAwareStrategy` | Beta 2 | Bounds candidates by distance, then breaks ties by longest-waiting Driver (`BecameAvailableAt`) |

Strategy selection is resolved through an `IMatchingStrategyFactory`, keyed by zone and time-of-day, so the dispatch service never depends on a concrete strategy.

### External Integrations

| System | Role | Boundary |
|---|---|---|
| Payment Provider | Captures rider charges, issues driver payouts | Integrated, not built; Rydex never stores raw card data |
| Maps / Routing Provider | Routing, ETA, distance calculation | Integrated, not built |
| Identity Provider | OAuth2/OIDC social login | Optional, in addition to email/phone authentication |
| AI/LLM Provider | Hosted LLM and embedding model for the RAG support assistant | Integrated, not built |

Each integration point requires an explicit degraded-mode behavior (e.g., a cached last-known ETA if routing is unavailable) rather than an unhandled failure — third-party outages are a named risk (see [Risk Register](#risk-register)).

## Architecture Decision Records

| ADR | Decision | Status |
|---|---|---|
| ADR-001 | Modular monolith for Alpha/Beta 1; Matching Engine and Fraud Service extracted as separate processes starting Beta 2 | Accepted |
| ADR-002 | EF Core for writes and simple reads; Dapper for trip-history and ops-dashboard read paths | Accepted |
| ADR-003 | Redis as primary system of record for live location, with SQL/PostGIS geography fallback under degradation | Accepted |
| ADR-004 | RabbitMQ + MassTransit for the event-driven messaging backbone | Accepted |
| ADR-005 | REST + JSON for both external and internal service-to-service traffic; gRPC deferred as a stretch-tier optimization | Accepted |
| ADR-006 | Polling background service for the transactional Outbox relay, not Change Data Capture | Accepted |
| ADR-007 | pgvector for RAG embeddings, not a dedicated vector database | Accepted |

Full context, consequences, and alternatives considered for each ADR are recorded in the domain and architecture documentation.

## API

Rydex exposes a resource-oriented REST API, versioned from day one under `/v1/`, defined contract-first in `api/openapi.yaml` (OpenAPI 3.0.3).

**Conventions:**

- `Idempotency-Key` header required on `POST /trips` — a retried request with the same key never creates a second Trip.
- Cursor-based pagination on `GET /trips/history` (`?cursor=`, `?limit=`) — never offset, since offset pagination breaks under concurrent inserts.
- RFC 7807 Problem Details standardized across every error response via a single reusable `Problem` schema.
- `202 Accepted` on `POST /trips` — matching runs asynchronously; the match result is pushed via the real-time channel, not returned in the response.
- `409 Conflict` on `acceptTrip` and `cancelTrip` — covers both the double-booking concurrency race and illegal state transitions.
- HATEOAS-light `_links` on the `Trip` resource, indicating which actions (`cancel`, `rate`, `accept`) are currently valid.
- Bearer JWT authentication on all endpoints.

**Endpoints:**

| Method | Path | Description |
|---|---|---|
| `POST` | `/trips` | Request a trip; creates a Trip in `Requested` status and begins matching asynchronously |
| `GET` | `/trips/history` | Cursor-paginated trip history for the authenticated Rider or Driver |
| `GET` | `/trips/{tripId}` | Trip details, including embedded Fare and `_links` |
| `POST` | `/trips/{tripId}/cancel` | Cancel a trip; always captures a reason |
| `POST` | `/trips/{tripId}/accept` | Driver accepts an outstanding match offer |
| `POST` | `/trips/{tripId}/rating` | Submit a rating for a completed trip |
| `POST` | `/drivers` | Driver onboarding |
| `GET` | `/drivers/{driverId}` | Get driver profile |
| `POST` | `/drivers/{driverId}/approve` | Admin approves a driver's onboarding |
| `POST` | `/drivers/{driverId}/suspend` | Admin suspends a driver |
| `POST` | `/riders` | Rider signup |
| `GET` | `/riders/{riderId}` | Get rider profile |
| `POST` | `/auth/login` | Email/phone login; returns access and refresh tokens |

> **Known gap:** a `POST /auth/refresh` endpoint is required by the refresh-token rotation requirement but is not yet defined in the v1 contract — tracked for resolution before Alpha implementation closes.

## Repository Structure

```
rydex/
├── Rydex.sln
├── .editorconfig
├── Directory.Build.props
├── global.json
├── src/
│   ├── Rydex.Api/                # ASP.NET Core Web API — controllers, hubs, middleware, DI wiring
│   ├── Rydex.Domain/              # Aggregates, value objects, domain events, enums — zero framework references
│   ├── Rydex.Application/         # MediatR commands/queries, vertical-slice features, pipeline behaviors
│   ├── Rydex.Infrastructure/      # EF Core, Dapper, repositories, messaging, caching, identity
│   ├── Rydex.Worker/               # Background services — Outbox relay
│   └── Rydex.MatchingEngine/       # Extracted matching strategies (populated starting Beta 2)
├── api/
│   └── openapi.yaml                # v1 OpenAPI contract — source of truth for the REST surface
├── infra/
│   └── docker-compose.yml          # PostgreSQL, Redis, RabbitMQ, Jaeger, Prometheus, Grafana, API
├── .github/
│   └── workflows/
│       └── ci.yml                  # restore → build → test → docker build
└── docs/
    ├── rydex-product-overview.md
    ├── rydex-requirements-and-risk.md
    ├── rydex-domain-and-architecture.md
    ├── rydex-api-contract.md
    └── rydex-engineering-execution-guide.md
```

Project references follow the Clean Architecture dependency rule: `Domain` depends on nothing; `Application` depends only on `Domain`; `Infrastructure` depends on `Domain` and `Application`; `Worker` and `Api` depend on `Application` and `Infrastructure`. `Api` never references `Infrastructure`'s concrete types directly outside dependency injection registration.

## Getting Started

### Prerequisites

- .NET SDK (version pinned in `global.json`)
- Docker and Docker Compose
- Git
- `dotnet-ef` CLI tool (`dotnet tool install --global dotnet-ef`)

### Local Setup

```bash
# Clone the repository
git clone <repository-url>
cd rydex

# Start local infrastructure — PostgreSQL, Redis, RabbitMQ, and the observability stack
docker compose -f infra/docker-compose.yml up -d

# Restore dependencies
dotnet restore

# Apply database migrations
dotnet ef database update \
  --project src/Rydex.Infrastructure \
  --startup-project src/Rydex.Api

# Run the API
dotnet run --project src/Rydex.Api
```

The API is versioned under `/v1/`. With the stack running locally, the OpenAPI contract at `api/openapi.yaml` can be loaded into Swagger UI, Postman, or any OpenAPI-compatible client to explore or mock the surface before backend implementation is complete.

### Configuration

Connection strings and environment-specific settings are read via the Options pattern from `appsettings.json` / `appsettings.Development.json` — no connection details are hardcoded. Local secrets for development use `dotnet user-secrets`; production secrets are sourced from Azure Key Vault (or the equivalent secret store for the deployment target).

## Testing

| Layer | Approach |
|---|---|
| Unit tests | Application-layer handlers tested against repository interfaces, without a real database |
| Integration tests | Testcontainers against a real PostgreSQL instance, explicitly covering the double-booking concurrency race |
| Contract tests | Consumer-driven contract tests against the OpenAPI surface |
| Load tests | k6 — realistic user-flow scripts (request → accept → location ping → complete), spike tests, and multi-hour soak tests |
| Mutation testing | Targeted at the matching engine and pricing logic, where a passing-but-weak suite is most dangerous |

```bash
dotnet test
```

## Observability

- **Logging:** Serilog, structured, with correlation-ID propagation across every request.
- **Tracing:** OpenTelemetry, exported to Jaeger — full request lifecycle from API through Matching Engine, Redis, RabbitMQ, and downstream consumers.
- **Metrics:** Custom OpenTelemetry metrics (`Rydex.trips.matched`, `Rydex.matching.latency_ms`, `Rydex.surge.active_zones`) scraped by Prometheus.
- **Dashboards and alerting:** Grafana, mapped one-to-one against the SLOs defined below — no dashboard exists without a corresponding, stated target.
- **Health checks:** `/health/live` and `/health/ready`, including database and external-dependency checks.

## Roadmap

| Milestone | What ships | What it proves |
|---|---|---|
| **Alpha** | Account creation, basic trip request/match, trip history, payment capture | The core loop works end to end |
| **Beta 1** | Live tracking, real-time status updates, driver presence | The product feels real-time |
| **Beta 2** | Batch matching, fairness-aware queueing, surge pricing, zone management | The marketplace logic is genuinely intelligent |
| **Beta 3** | Fraud/anomaly detection, AI support assistant, smart escalation | Trust & safety and support are production-credible |
| **Release Candidate** | Full observability buildout, load testing, hardened testing pass, CI/CD maturity | Scale claims are backed by evidence, not assumed |
| **Showcase Release** | UX polish, full documentation, demo script | The system is ready to present externally |

## Success Metrics

| Metric | What it measures |
|---|---|
| Time-to-match | How fast riders are connected to a driver |
| Match acceptance rate | Whether offered matches are genuinely good matches, not just fast |
| Cancellation rate | Aggregate friction/dissatisfaction signal |
| ETA accuracy | Trustworthiness of the live-tracking experience |
| Fraud flag precision | Whether the fraud system is useful signal or noise |
| AI support deflection rate | How much support load the AI assistant genuinely absorbs |
| System uptime / p95 latency | Whether the platform holds up under real conditions |

## Service Level Objectives

| NFR | SLO | Verified by |
|---|---|---|
| Matching speed | p95 < 2s, p99 < 3s | k6 load test + Grafana dashboard |
| Real-time update latency | p95 location push < 500ms | SignalR instrumentation + Prometheus histogram |
| System availability | 99.5% over rolling 30 days | Uptime monitor + alerting |
| Scalability | Target 500 concurrent active trips (planning placeholder); actual measured ceiling reported | k6 soak and spike test report |
| Data integrity | Zero loss or duplication incidents on Trip/Payment records | Testcontainers integration tests covering concurrency races |
| Security & privacy | 100% HTTPS/HSTS, zero raw card storage, zero PII in logs, 100% resource-level authorization enforcement | Automated auth tests, log audits, TLS configuration checks |

The Scalability SLO's "500" figure is an explicit planning placeholder for designing the Release Candidate load test — it is not a committed or achieved number until that test runs.

## Risk Register

The full risk register (13 entries, product-level and engineering-discovered) is maintained in the requirements and risk documentation. Top-line risks:

| ID | Risk | Impact | Status |
|---|---|---|---|
| R-01 | Small team, bounded scope — single-region simulation, not commercial scale | Medium (reviewer perception) | Mitigated by stated scope |
| R-02 | Third-party dependency outages (maps, payments, AI providers) | High | Open — degraded-mode behavior required per integration before Release Candidate |
| R-04 | Location data sensitivity — privacy and access control | High | Mitigated via Security & Privacy SLO |
| R-05 | Redis persistence-mode misconfiguration for the fairness queue | High | Open — ADR required before Beta 2 |
| R-09 | Graceful-degradation availability claim is conditional on unconfirmed fallback paths | High | Open — requires chaos/failure-injection testing |
| R-10 | AI support assistant's policy-content prerequisite has no assigned owner | Medium-High | **Open — unassigned, most urgent open item** |

## Documentation

Full project documentation is maintained alongside this repository:

| Document | Contents |
|---|---|
| Product Overview | Vision, strategy, product glossary, personas, feature roadmap, success metrics |
| Requirements and Risk | Acceptance criteria (traceability matrix), SLO definitions, risk register |
| Domain and Architecture | Domain glossary, aggregates, bounded contexts, ADR-001–007, C4 diagrams, database schema, matching engine design |
| API Contract | Design rationale and the complete OpenAPI v1 specification |
| Engineering Execution Guide | The literal, ordered sequence of engineering steps from an empty repository through Showcase Release |

## Contributing

- Branch protection is enabled on `main`, requiring the CI workflow (`restore → build → test → docker build`) to pass before merge.
- Every pull request should reference the acceptance criteria or ADR it implements where applicable.
- New domain terminology should be reconciled against the domain glossary before introduction into code, comments, or PR descriptions — in particular, `Matched` (an offer is outstanding) must never be used to mean `DriverEnRoute` (the driver has accepted).
- Dependabot is active for automated dependency updates and security alerts.

## License

MIT LICENSE
