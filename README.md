# RBAC Dashboards

Reference implementation of a role-based access control dashboard system — Admin, Operations and
Consultant roles over a shared task management domain.

The focus is deliberately **depth of security over breadth of features**: method-level
authorization, immutable audit trails, per-topic WebSocket authorization, and a negative
authorization test suite that proves each boundary holds.

<!-- Add once CI exists:
[![CI](https://github.com/erikestr/rbac-dashboards/actions/workflows/ci.yml/badge.svg)](https://github.com/erikestr/rbac-dashboards/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
-->

> [!NOTE]
> **Intentional scope.** Configurable workflow engines, full multi-instance real-time fan-out and
> advanced report exports are documented in [`docs/adr/`](docs/adr/) with estimates rather than
> implemented. This project demonstrates the security foundation of an RBAC system, not a complete
> feature catalogue. Each decision records the tradeoff and the cost to add it.

---

## Demo

|                 |                        |
| --------------- | ---------------------- |
| **Live**        | _TODO: Cloud Run URL_  |
| **Walkthrough** | _TODO: 2-minute video_ |

**Demo credentials**

| Role       | Email                   | Password |
| ---------- | ----------------------- | -------- |
| Admin      | `admin@demo.local`      | _TODO_   |
| Operations | `ops@demo.local`        | _TODO_   |
| Consultant | `consultant@demo.local` | _TODO_   |

> The demo database resets nightly. Feel free to break things.

<!-- TODO: screenshots — one per dashboard, plus the audit log viewer -->

---

## Why this exists

Most RBAC implementations fail in the same three places, and all three are invisible in a
screenshot:

1. **The permission check lives in the frontend.** The menu item is hidden, the endpoint is open.
2. **The query is missing its scope filter.** `GET /api/tasks/42` returns task 42 regardless of
   who owns it — [BOLA](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/),
   number one in the OWASP API Security Top 10.
3. **The token is never really verified.** Expiry is skipped, or checked once at connection time
   and never again on a long-lived session.

This codebase is built to demonstrate that those three failures are absent, and to make that
demonstrable rather than asserted — see [Security](#security).

---

## Security

The part worth reading. Everything here is enforced server-side and covered by tests.

### Authentication

- JWT access tokens (short-lived) + refresh tokens with **rotation and reuse detection** — a
  replayed refresh token invalidates the entire family
- BCrypt password hashing with a tuned work factor
- Password recovery via single-use, expiring tokens
- No sensitive claims in the JWT payload

### Authorization

- **Method-level** `@PreAuthorize` on the service layer, not route-level guards on the controller
- **Permission-based authorities** (`task:delete`, `user:manage`) rather than role string checks —
  roles are compositions of permissions, so adding a role requires no code change
- Every scoped query filters by ownership at the repository layer via JPA Specifications
- Privilege escalation closed: the profile update endpoint cannot alter roles or permissions
- DTO boundaries throughout — JPA entities are never exposed, closing mass assignment

### WebSocket

- Handshake authentication (no `Authorization` header available at the transport layer)
- **Per-topic subscription authorization** — a Consultant subscribing to `/topic/admin/*` is
  rejected at the broker, not filtered client-side
- Mid-session token expiry handling on long-lived connections
- Reconnection with state resync

### Audit

- Append-only audit log — no `UPDATE`, no `DELETE`, enforced at the database level
- Records actor, action, target, timestamp and source address
- Passwords, tokens and unnecessary PII are never logged
- Paginated Admin viewer with filtering

### Hardening

- Rate limiting via Bucket4j on authentication endpoints
- OWASP security headers, explicit CORS origins (never `*`)
- Bean Validation on every inbound DTO
- Dependency scanning: Dependabot + Trivy in CI
- OWASP ZAP baseline scan — [report](docs/security/zap-baseline.md)

### Evidence

- **[Threat model](docs/security/threat-model.md)** — what is protected, from whom, what happens on failure
- **[`docs/security/`](docs/security/)** — one document per concept, written from scratch rather than generated
- **Negative authorization test suite** — see [Testing](#testing)

---

## Stack

| Layer      | Choice                          | Rationale                                                                                            |
| ---------- | ------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Backend    | Java 25 (LTS) + Spring Boot 4.1 | Boot 3.x reached end of OSS support 30 Jun 2026. 4.1 receives free security patches through Jul 2027 |
| Security   | Spring Security + JWT           | Method-level authorization that is genuinely auditable                                               |
| Database   | PostgreSQL 16                   | Row-level scoping, JSONB settings, indexed reporting                                                 |
| Migrations | Flyway                          | Versioned, reproducible schema                                                                       |
| Frontend   | React 19 + TypeScript + Vite    | Typed end to end                                                                                     |
| UI         | Tailwind + shadcn/ui            | Consistent responsive system from day one                                                            |
| Charts     | Recharts                        | KPIs without additional weight                                                                       |
| Real-time  | WebSocket (STOMP)               | Notifications only — see [ADR 002](docs/adr/002-polling-over-full-realtime.md)                       |
| Files      | Cloudflare R2 + signed URLs     | Uploads never transit the backend                                                                    |
| Email      | Resend                          | Real deliverability                                                                                  |
| Infra      | Docker + Cloud Run + Neon       | Scale-to-zero on both compute and database                                                           |
| CI/CD      | GitHub Actions                  | Tests, scans and automatic deploys                                                                   |

---

## Features by role

### Admin
- User management — create, edit, deactivate
- Role and permission assignment
- Task assignment to Operations and Consultants
- Progress monitoring across the organization
- KPI dashboard with charts
- Audit log viewer
- CSV export

### Operations
- Scoped task queue with priorities and due dates
- Status updates with optimistic locking
- Assignment and escalation
- Comment threads
- Search, filter and sort

### Consultant
- Personal task list with deadlines
- Progress updates
- Document uploads via signed URLs
- Comment threads
- Personal performance view

---

## Project structure

```
backend/src/main/java/dev/erikestr/rbac/
├── auth/           # login, JWT issuance, refresh rotation with reuse detection
├── user/           # user lifecycle, profile
├── role/           # roles, permissions, assignment
├── task/           # task domain, assignment, escalation, state transitions
├── comment/        # threads on tasks
├── notification/   # in-app, email, WebSocket delivery
├── audit/          # append-only audit trail
└── shared/
    ├── config/         # security, WebSocket, CORS
    ├── security/       # filters, permission evaluator, STOMP interceptor
    ├── exception/      # global error handling
    └── specification/  # base scoping filter — the layer that prevents BOLA
```

Organized by feature rather than by layer: each package owns its controller, service, repository
and domain. The question *"does this feature scope its queries correctly?"* is answered by reading
one directory.

`shared/specification/` is deliberately its own package. It holds the ownership filter every scoped
query composes with, and keeping it visible as shared infrastructure is what stops each feature
from reinventing — and forgetting — it.

---

## Getting started

**Requirements:** Docker, Docker Compose, JDK 25, Node 22+

```bash
git clone https://github.com/erikestr/rbac-dashboards.git
cd rbac-dashboards

cp .env.example .env                        # no real secrets needed for local
docker compose up -d                        # PostgreSQL + MailHog

cd backend  && ./mvnw spring-boot:run       # → http://localhost:8080
cd frontend && npm install && npm run dev   # → http://localhost:5173
```

Flyway migrates and seeds realistic demo data on first boot. API documentation is served at
`http://localhost:8080/swagger-ui.html`.

---

## Testing

```bash
./mvnw test                        # unit + integration (Testcontainers)
./mvnw test -Dgroups=authorization # authorization suite only
cd web && npm test                 # frontend (Vitest)
```

The **negative authorization suite** is the centrepiece. Rather than asserting that permitted
actions succeed, it asserts that forbidden ones fail — at the endpoint, not the UI:

```java
@Test void consultantCannotAccessAdminEndpoints()
@Test void consultantCannotReadTaskOutsideTheirScope()
@Test void operationsCannotAssignRoles()
@Test void userCannotEscalateOwnRoleViaProfileUpdate()
@Test void expiredAccessTokenIsRejected()
@Test void reusedRefreshTokenInvalidatesTokenFamily()
@Test void consultantCannotSubscribeToAdminWebSocketTopic()
@Test void auditLogRejectsUpdateAndDelete()
```

Every endpoint has at least one negative case. _TODO: coverage numbers once the suite exists._

---

## Documentation

| Path                                   | Contents                                                            |
| -------------------------------------- | ------------------------------------------------------------------- |
| [`docs/adr/`](docs/adr/)               | Architecture decisions, including everything deliberately not built |
| [`docs/security/`](docs/security/)     | One document per security concept, in plain language                |
| [`docs/ai-usage.md`](docs/ai-usage.md) | What was delegated to AI, what was written by hand, and why         |
| [`docs/cost.md`](docs/cost.md)         | Running infrastructure cost and how it is kept bounded              |

---

## Intentional scope

Five capabilities from the original brief were deliberately excluded. Each has an ADR recording
the decision, the tradeoff and the estimated cost to add it.

| Not built                                                                       | Reason                                                                                                     | Cost to add  |
| ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ------------ |
| [Configurable workflow engine](docs/adr/001-no-workflow-engine.md)              | Requires a real client's process definition; building one in a vacuum builds the wrong one                 | 6+ weeks     |
| [Full real-time fan-out](docs/adr/002-polling-over-full-realtime.md)            | Below ~50 concurrent users, 30s polling is indistinguishable and saves ~$50/mo in always-on infrastructure | ~25h + Redis |
| [Materialized views for analytics](docs/adr/003-plain-queries-for-analytics.md) | Identical output at demo data volume; premature optimization                                               | ~10h         |
| [Excel / PDF export](docs/adr/004-csv-only-export.md)                           | Same pattern as the implemented CSV export — no additional capability demonstrated                         | ~6h          |
| [Multi-tenancy](docs/adr/005-single-tenant.md)                                  | A day-one architecture decision that cannot be made without a client                                       | Significant  |

---

## Operating cost

This runs on **~$0/month**. Public repo (unmetered GitHub Actions), Neon serverless Postgres and
Cloud Run both scaling to zero, Cloudflare R2 with no egress fees, and hard budget alerts at $1,
$5 and $10. Full breakdown in [`docs/cost.md`](docs/cost.md).

Managed instances that never scale to zero — Cloud SQL in particular — are what turn a portfolio
project into a recurring bill. That tradeoff is documented rather than discovered.

---

## License

MIT — see [LICENSE](LICENSE).
