# Microservices

> **Microservice:** An independent, task-focused service that owns its own business context and defines its own physical boundary.

-> Decomposing a large application into small, independently deployable services.

---

## What Are Microservices

A big application is divided into smaller, independent services. Each service deals with its own context of business rules and defines its own physical boundary. Microservices are highly scalable: instances can be replicated to handle environments with large volumes of incoming and outgoing data.

-> Logical boundaries translate directly into physical boundaries, making the move from modular monolith to microservices a natural approach.

---

## Benefits

**Independent deployability:** each microservice can be deployed on its own without touching other services.

**Organizational autonomy:** one team can be assigned to each bounded context, enabling parallel work across teams.

**Fault isolation:** a less reliable service can be decoupled and isolated to contain the blast radius of failures.

**Scalability:** each service scales independently based on demand, with a load balancer placed in front of high-availability instances.

---

## Modular Monolith as a Stepping Stone

> "Well-defined, in-process modules (modules) can be an excellent stepping stone to out-of-process components (services)."
