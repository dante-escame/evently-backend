# Unit Tests

> **Domain Business Rules:** The base layer of the test pyramid, focused on verifying that the Domain model behaves correctly in complete isolation, with no infrastructure dependencies.

-> Catching business rule violations fast, before anything ever hits the database or the network.

---

## What Unit Tests Cover

Unit tests in this codebase target the **Domain layer**: Aggregates, Entities, Value Objects, and Domain Services. These are the types that encode the core business rules of the application, and they are the most important things to keep correct.

Each test exercises a single behavior in isolation:

- A valid state transition on an Aggregate succeeds and raises the expected Domain Event.
- An invariant violation is rejected with the correct error.
- A Value Object rejects invalid input and accepts valid input.

Infrastructure concerns such as databases, message brokers, or HTTP clients are never involved.

---

## Libraries

| Library | Role |
|---|---|
| **Bogus** | Generates realistic fake data for test inputs (names, emails, dates, GUIDs, etc.) |
| **FluentAssertions** | Provides a readable, expressive assertion API for verifying outcomes |

### Bogus

Bogus is a .NET library for generating fake but realistic data. It avoids hardcoded magic strings in tests and makes it easy to cover a wide variety of input shapes without writing data builders by hand.

```csharp
var faker = new Faker();

string name = faker.Name.FullName();
string email = faker.Internet.Email();
Guid id = faker.Random.Guid();
```

### FluentAssertions

FluentAssertions replaces plain `Assert` calls with a natural language chain that makes the intent of each assertion immediately clear.

```csharp
result.IsSuccess.Should().BeTrue();
result.Error.Should().Be(TicketErrors.AlreadyCanceled);
domainEvents.Should().ContainSingle().Which.Should().BeOfType<TicketCanceledDomainEvent>();
```

---

*(...to be continued)*
