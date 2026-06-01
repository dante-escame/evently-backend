> "Choose microservices for the benefits, not because your monolithic codebase is a mess."
> — Simon Brown

> "You shouldn't start a new project with microservices, even if you're sure your application will be big enough to make it worthwhile."
> — Martin Fowler

## Fallacies of Distributed Computing
- The network is not reliable.
- Latency is not zero.
- Bandwidth is not infinite.
- The network is not secure.
- Topology does indeed change.
- There is more than one administrator.
- Transport cost is not zero.
- The network is not homogeneous.

By starting with a monolith, it turns out to be easier to define application boundaries.
Microservices with poor boundaries elevate the cost of maintenance and new features. The mess will always get bigger.

What is the solution then?

With the physical architecture of a monolith and the logical architecture of microservices, we can merge the benefits of each approach.

---

# Modular Monolith
A software design approach in which a monolith is designed with an emphasis on interchangeable (and potentially reusable) modules.
Modules are separate parts that when combined form a complete whole.
In this architecture, modules should be treated as self-contained, separated applications (even if they are in the same GIT repository).

> "I personally don't like the name 'Modular Monolith' because of the stigma built in the development market about Monolithic Architectures, mainly around the distributed computing enjoyers. This architecture approach is great for starting up projects."

A modular monolith backend for event management, built to explore and apply patterns specific to this architectural style.

## Stack
- .NET 8 / ASP.NET Core
- PostgreSQL with EF Core + Dapper (read queries)
- Redis
- Keycloak (identity & authorization)
- MassTransit (messaging)
- MediatR (CQRS)
- Quartz.NET (background job scheduling)
- FluentValidation (input validation)
- Serilog + Seq (structured logging)
- Docker + Docker Compose
- xUnit + Bogus + FluentAssertions + Testcontainers (testing)

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
