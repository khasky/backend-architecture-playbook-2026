# Backend Architecture Playbook 2026

Practical backend architecture guide for APIs, services, boundaries, data access, testing, and system evolution.

> *If I were setting a backend standard for a product team today, I would optimize for four things first: explicit request flow, strict model boundaries, predictable error handling, and contracts that machines can validate.*

---

## Table of Contents

- [Backend Architecture Playbook 2026](#backend-architecture-playbook-2026)
  - [Table of Contents](#table-of-contents)
  - [Why this exists](#why-this-exists)
  - [The defaults I'd reach for first](#the-defaults-id-reach-for-first)
  - [Request flow](#request-flow)
  - [Models and boundaries](#models-and-boundaries)
    - [The boundary model split](#the-boundary-model-split)
    - [Why this matters](#why-this-matters)
  - [Middleware and decorators](#middleware-and-decorators)
    - [Good candidates for middleware](#good-candidates-for-middleware)
    - [Good candidates for decorators or controller-level annotations](#good-candidates-for-decorators-or-controller-level-annotations)
    - [The defaults I would keep](#the-defaults-i-would-keep)
  - [Error handling](#error-handling)
    - [The model I would use](#the-model-i-would-use)
    - [A practical mapping](#a-practical-mapping)
    - [Why I would avoid status-code leakage](#why-i-would-avoid-status-code-leakage)
  - [OpenAPI-first contracts](#openapi-first-contracts)
    - [What I would treat as the contract baseline](#what-i-would-treat-as-the-contract-baseline)
    - [Why OpenAPI belongs here](#why-openapi-belongs-here)
    - [Practical rules I would adopt](#practical-rules-i-would-adopt)
  - [Persistence and repositories](#persistence-and-repositories)
    - [What repositories should do](#what-repositories-should-do)
    - [What repositories should not do](#what-repositories-should-not-do)
    - [Data access defaults](#data-access-defaults)
  - [Testing strategy](#testing-strategy)
    - [Minimum useful testing stack](#minimum-useful-testing-stack)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [Database-backed tests](#database-backed-tests)
    - [What I would insist on](#what-i-would-insist-on)
  - [A concrete request example](#a-concrete-request-example)
  - [Things I would avoid](#things-i-would-avoid)
  - [Suggested repository structure](#suggested-repository-structure)
  - [References and inspiration](#references-and-inspiration)
    - [Official and high-signal references](#official-and-high-signal-references)
    - [Similar or adjacent GitHub repositories](#similar-or-adjacent-github-repositories)
  - [License](#license)

---

## Why this exists

A lot of backend codebases do not fail because the language is wrong or the framework is weak. They fail because responsibilities leak everywhere:

- controllers start doing business logic;
- services start speaking HTTP;
- repositories become mini business layers;
- ORM models become the app's entire mental model;
- error handling turns into scattered `try/catch` folklore.

This README is meant to be a better default.

It takes the raw notes from this repository and turns them into a production-minded backend guide that is easier to share, teach, and evolve.

---

## The defaults I'd reach for first

If I were starting a service today, I would usually default to something close to this:

- **Flow:** controller -> service -> repository -> data store or external gateway
- **Boundary models:** DTOs at the edge, domain models in business logic, persistence models in the data layer
- **Validation:** request validation at the boundary, before business logic runs
- **Auth:** middleware or decorators, never hand-written in every controller method
- **Observability:** request ID on every request and response, structured logs, tracing hooks
- **Errors:** custom domain/application exceptions + one global error handler
- **API contract:** OpenAPI as the source of truth for request/response shape
- **Persistence:** repositories own database access; services do not speak SQL/ORM directly
- **Tests:** unit tests for services/repositories, integration tests for controllers, database-backed tests in containers

That stack is not fancy. That is the point.

---

## Request flow

The backend flow I would defend in code review is:

1. **Controller**
   - receives the request;
   - validates and normalizes input;
   - delegates to the service layer;
   - maps domain output back into response DTOs.

2. **Service**
   - owns business rules;
   - orchestrates repositories and gateways;
   - works with domain models, not HTTP details.

3. **Repository**
   - owns data access;
   - translates between persistence models and domain models;
   - contains query logic, not product decisions.

4. **External gateways**
   - wrap third-party APIs, queues, payment providers, internal service clients;
   - should have explicit retry behavior where retries are safe and intended.

A good heuristic: if a function needs to know HTTP status codes, it probably belongs near the edge. If it needs to know business rules, it probably belongs in the service layer. If it needs to know tables, indexes, or ORM details, it belongs in the repository layer.

---

## Models and boundaries

One of the strongest ideas in the source notes is the distinction between **request/response models**, **domain models**, and **database models**.

That is worth keeping.

### The boundary model split

- **DTOs / request-response models**
  - shape the transport contract;
  - reflect what the API accepts and returns;
  - are not the same thing as internal entities.

- **Domain models**
  - represent business concepts;
  - exist to make business logic readable and stable;
  - should not be forced to mirror database tables or API payloads exactly.

- **Persistence models / ORM models / DAOs**
  - exist to talk to storage;
  - may include persistence-only concerns;
  - should not quietly become your public API.

### Why this matters

When one model tries to do everything, three problems show up quickly:

- database decisions leak into product code;
- API shape gets trapped by storage shape;
- refactors become expensive because every layer depends on the same object.

Keeping models distinct costs a little more up front and pays back every time the system evolves.

---

## Middleware and decorators

Cross-cutting concerns should feel boring, centralized, and repeatable.

### Good candidates for middleware

- request IDs;
- authentication bootstrap;
- tracing;
- metrics hooks;
- rate limiting;
- body size limits;
- common request logging.

### Good candidates for decorators or controller-level annotations

- request validation;
- auth requirements;
- analytics hooks;
- method-level logging;
- dependency injection wiring.

### The defaults I would keep

- `X-Request-ID` or equivalent request correlation header;
- validation decorators for query params, path params, and body;
- auth decorators for API key and bearer-token protected routes;
- structured method logging that can capture args, result metadata, and exceptions safely.

The important rule is not "use decorators" The rule is "do not repeat policy by hand in every endpoint".

---

## Error handling

This is the part most teams under-design.

### The model I would use

- services and repositories throw **custom application errors**;
- those errors do **not** know about HTTP transport;
- one **global error handler** maps those exceptions into public HTTP responses.

### A practical mapping

- `ValidationError` -> `400 Bad Request`
- `AuthenticationError` -> `401 Unauthorized`
- `AuthorizationError` -> `403 Forbidden`
- `NotFoundError` -> `404 Not Found`
- `ConflictError` -> `409 Conflict`
- uncaught or unknown errors -> `500 Internal Server Error`

### Why I would avoid status-code leakage

When services throw transport-aware results like "return 404 here" or "respond 422 from repository", the architecture gets inverted. Business logic starts depending on delivery semantics. That makes reuse harder and testing noisier.

I would also avoid overusing response codes purely because they exist. A smaller, well-understood set is usually easier for clients and maintainers to reason about.

---

## OpenAPI-first contracts

OpenAPI is one of the easiest ways to make a backend feel more real, even before it grows large.

### What I would treat as the contract baseline

- endpoint paths and methods;
- request schemas;
- response schemas;
- auth scheme definitions;
- error response shapes;
- examples for important edge cases.

### Why OpenAPI belongs here

An OpenAPI document gives both humans and tools a language-agnostic interface description. That makes it useful for:

- validation;
- generated models;
- generated clients;
- documentation;
- contract review;
- integration across multiple languages.

### Practical rules I would adopt

- use **plural nouns** for collection resources where it makes sense;
- keep request and response schemas explicit;
- generate typed clients or models where that reduces drift;
- let the contract drive validation, not the other way around.

---

## Persistence and repositories

Repositories are one of the best places to win back clarity.

### What repositories should do

- encapsulate persistence details;
- handle ORM/query-builder specifics;
- translate persistence records into domain models;
- own transaction and rollback behavior when appropriate.

### What repositories should not do

- decide product policy;
- format HTTP responses;
- call unrelated services just because it is convenient;
- become a dumping ground for business logic.

### Data access defaults

- use ORM models or query abstractions for database access where they genuinely help;
- wrap write operations in transaction-aware error handling;
- rollback explicitly when the transaction model requires it;
- keep repository methods named after business intent, not just SQL verbs.

---

## Testing strategy

The source material is very clear here, and I think it is right.

### Minimum useful testing stack

#### Unit tests

Use them for:

- service logic;
- domain rules;
- repository behavior that can be validated in isolation;
- utility logic with branching behavior.

#### Integration tests

Use them for:

- controller -> service -> repository flow;
- real request/response behavior;
- validation, auth, and serialization boundaries;
- database interaction with realistic setup.

#### Database-backed tests

If repository behavior matters, test against a real database or a close-enough ephemeral environment. Containers are often the cleanest tradeoff.

### What I would insist on

- new behavior gets tests, not only bugfix patches after incidents;
- controller integration tests cover the full request lifecycle;
- database migrations used in tests are realistic enough to catch schema drift early.

---

## A concrete request example

```txt
HTTP request
  -> middleware attaches request_id
  -> auth middleware/decorator resolves caller
  -> validation decorator validates body/query/path
  -> controller maps input DTO -> domain request
  -> service executes business use case
  -> repository loads/saves persistence models
  -> service returns domain result
  -> controller maps domain result -> response DTO
  -> global error handler maps exceptions if needed
  -> HTTP response includes request_id
```

And in pseudocode:

```ts
async function addUser(req) {
  const request = UserRequestDto.parse(req.body)
  const createdUser = await userService.create(request.toDomain())
  const result = UserResponseDto.fromDomain(createdUser)

  return {
    statusCode: 200,
    body: {
      request_id: req.requestId,
      result,
    },
  }
}
```

The controller is thin. The service is meaningful. The response shape is explicit. That is the general goal.

---

## Things I would avoid

- controllers that know SQL or ORM details;
- repositories returning raw HTTP responses;
- services throwing framework-specific response objects;
- one giant `User` model reused for transport, domain, and persistence;
- silent retries without idempotency thinking;
- auth logic copy-pasted into handlers;
- analytics calls that can break business flows because they are not isolated;
- tests that mock everything and therefore verify nothing important.

---

## Suggested repository structure

```txt
src/
  api/
    controllers/
    dto/
    middlewares/
    decorators/
    presenters/
  application/
    services/
    errors/
    ports/
  domain/
    models/
    value-objects/
    policies/
  infrastructure/
    repositories/
    orm/
    gateways/
    clients/
    logging/
    tracing/
  openapi/
    openapi.yaml
  tests/
    unit/
    integration/
```

This is not the only valid structure. It is simply a structure that makes the boundaries visible.

---

## References and inspiration

### Official and high-signal references

- [OpenAPI Specification](https://github.com/OAI/OpenAPI-Specification)
- [Microsoft REST API design guidance](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [Microsoft REST API Guidelines repository](https://github.com/microsoft/api-guidelines)

### Similar or adjacent GitHub repositories

- [Backend-Architecture](https://github.com/amanmdave/Backend-Architecture)
- [Awesome Software Architecture](https://github.com/mehdihadeli/awesome-software-architecture)
- [API Best Practices](https://github.com/saifaustcse/api-best-practices)

---

## License

MIT is a sensible default for a repository like this, but choose the license that fits how you want others to reuse the material.
