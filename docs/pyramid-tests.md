# Pyramid Tests

> **Layered Test Strategy:** A model for structuring automated tests across three layers, each with a distinct scope, speed, and cost, so the suite stays fast, reliable, and meaningful.

-> Maximizing confidence while minimizing execution cost.

---

## The Three Layers

```
        /\
       /  \
      / E2E\
     /      \
    /Integrat\
   /          \
  /    Unit    \
 /--------------\
```

| Layer | Scope | Speed | Cost |
|---|---|---|---|
| **Unit** | Single class or method in isolation | Very fast | Very low |
| **Integration** | Multiple components interacting (DB, bus, DI) | Medium | Medium |
| **E2E** | Full system from HTTP request to response | Slow | High |

The pyramid shape reflects the **ideal test count ratio**: many unit tests at the base, fewer integration tests in the middle, and only a handful of end-to-end tests at the top.

---

## Why This Shape?

Unit tests are cheap to write and run in milliseconds, so they should cover every business rule and edge case. Integration tests verify that the wiring between components actually works, but spinning up infrastructure makes them slower. E2E tests give the highest confidence but are the most expensive to run and maintain, so they focus only on critical paths.

-> More tests at the base means faster feedback and lower CI cost without sacrificing confidence.

---

*(...to be continued)*
