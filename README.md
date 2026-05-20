# Evently

A modular monolith backend for event management, built to explore and apply patterns specific to this architectural style.

## Stack
- .NET 8 / ASP.NET Core
- PostgreSQL with EF Core
- Redis
- Keycloak (identity & authorization)
- MassTransit (messaging)
- Serilog + Seq (structured logging)
- Docker + Docker Compose

## Architecture
Single deployable unit composed of four self-contained modules, each structured in Domain, Application, Infrastructure, and Presentation layers:

- `src/Modules/Events` — event creation and publishing
- `src/Modules/Ticketing` — ticket purchase and management
- `src/Modules/Attendance` — attendance tracking
- `src/Modules/Users` — user registration and identity sync
- `src/Common/` — shared cross-cutting concerns (domain primitives, application abstractions, infrastructure utilities)
- `src/API/Evently.Api` — single API host wiring all modules together

## Concepts Covered
| Topic | Description |
| --- | --- |
| Modular Monolith | Single deployable unit with strict module isolation and enforced boundaries |
| Module Boundaries | Clear separation via architecture tests; no cross-module direct references |
| Module Communication | Sync (in-process interfaces) and async (domain events over MassTransit) |
| Auth with Keycloak | Token-based authentication and RBAC authorization via an external identity provider |
| Outbox & Inbox Patterns | Reliable at-least-once message delivery and idempotent consumers |
| Event-Driven Architecture | Domain events and integration events driving side effects across modules |
| Testing | Unit tests per module and integration tests against real infrastructure |

## Documentation
- Architecture notes, pattern breakdowns, and flow diagrams live in `docs/`.
