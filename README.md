# Backend Architecture Playbook 2026

Practical backend architecture guide for APIs, services, boundaries, data access, testing, and system evolution.

> *If I were setting a backend standard for a product team today, I would optimize for four things first: explicit request flow, strict model boundaries, predictable error handling, and contracts that machines can validate.*

---

## Table of Contents

- [Backend Architecture Playbook 2026](#backend-architecture-playbook-2026)
  - [Table of Contents](#table-of-contents)
  - [Why this exists](#why-this-exists)
  - [Companion playbooks](#companion-playbooks)
  - [The defaults I'd reach for first](#the-defaults-id-reach-for-first)
  - [Request flow](#request-flow)
  - [Models and boundaries](#models-and-boundaries)
    - [The boundary model split](#the-boundary-model-split)
    - [Why this matters](#why-this-matters)
  - [Cross-cutting concerns at the HTTP edge](#cross-cutting-concerns-at-the-http-edge)
    - [Good candidates for global pipeline steps](#good-candidates-for-global-pipeline-steps)
    - [Good candidates for route-scoped setup](#good-candidates-for-route-scoped-setup)
    - [The defaults I would keep](#the-defaults-i-would-keep)
  - [Error handling](#error-handling)
    - [The model I would use](#the-model-i-would-use)
    - [A practical mapping](#a-practical-mapping)
    - [Error payloads for TypeScript clients](#error-payloads-for-typescript-clients)
    - [Why I would avoid status-code leakage](#why-i-would-avoid-status-code-leakage)
  - [OpenAPI-first contracts](#openapi-first-contracts)
    - [What I would treat as the contract baseline](#what-i-would-treat-as-the-contract-baseline)
    - [Why OpenAPI belongs here](#why-openapi-belongs-here)
    - [Practical rules I would adopt](#practical-rules-i-would-adopt)
    - [Sharing the contract with a React client](#sharing-the-contract-with-a-react-client)
    - [Monorepos and a shared contract package](#monorepos-and-a-shared-contract-package)
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

## Companion playbooks

These repositories form one playbook suite:

- [Frontend Architecture Playbook 2026](https://github.com/khasky/frontend-architecture-playbook-2026) — React app structure, testing, and how the UI consumes APIs
- [DevOps Delivery Playbook 2026](https://github.com/khasky/devops-delivery-playbook-2026) — CI lanes, staging, and release habits for the same repos
- [Best of JavaScript 2026](https://github.com/khasky/best-of-javascript-2026) — curated tools that match the defaults below

---

## The defaults I'd reach for first

If I were starting a service today, I would usually default to something close to this:

- **Flow:** controller -> service -> repository -> data store or external gateway
- **Boundary models:** DTOs at the edge, domain models in business logic, persistence models in the data layer
- **Validation:** request validation at the boundary, before business logic runs
- **Auth:** enforced through the shared HTTP pipeline (middleware, hooks, `preHandler`, or composed wrappers), never hand-written inside every route handler
- **Observability:** request ID on every request and response, structured logs, tracing hooks
- **Errors:** custom domain/application exceptions + one global error handler
- **API contract:** OpenAPI as the source of truth for request/response shape
- **Persistence:** repositories own database access; services do not speak SQL/ORM directly
- **Tests:** **[Vitest](https://vitest.dev/)** for unit and integration-style tests in new TypeScript services; **Jest** remains fine in large legacy codebases; **Playwright** (or similar) for browser-level checks owned by the frontend repo; database-backed tests in containers where persistence behavior matters

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

## Cross-cutting concerns at the HTTP edge

Cross-cutting concerns should feel boring, centralized, and repeatable.

The same ideas show up under different names: **Express** leans on **middleware**; **Fastify** uses **hooks** (`onRequest`, `preHandler`, …) and **plugins** to package them; **Hono** uses **middleware** on the app and on routes. Below, "pipeline" means whatever runs around your handler in your chosen stack.

### Good candidates for global pipeline steps

- request IDs;
- authentication bootstrap (parsing tokens, attaching a principal when cheap);
- tracing;
- metrics;
- rate limiting;
- body size limits;
- common request logging.

### Good candidates for route-scoped setup

Things you want **declared when the route is registered**, not reimplemented inside the handler body:

- request validation for query, path, and body (for example: schema on the route, a validation middleware in the chain, or a small wrapper factory);
- auth requirements for a path prefix or specific routes;
- analytics or audit that should run only for certain endpoints;
- composing the handler with shared dependencies (factory functions, thin wrappers) so the handler stays a thin entrypoint.

### The defaults I would keep

- `X-Request-ID` or equivalent request correlation header;
- validation enforced at the route boundary (schema + shared validator, or middleware in the chain), so handlers receive already-checked input;
- auth enforced the same way for protected routes (shared guard middleware, hook, or `preHandler`-style step), including API key and bearer-token flows;
- structured logging around handlers that can capture safe metadata and errors without leaking secrets.

The important rule is not which keyword your framework uses. The rule is "do not repeat policy by hand in every endpoint".

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

### Error payloads for TypeScript clients

Your React app will branch on status codes and parse bodies. Agree on a **small, stable JSON shape** for error responses and describe it in OpenAPI (and in the [frontend playbook](https://github.com/khasky/frontend-architecture-playbook-2026) the client should use the same assumptions).

I would standardize on something predictable along these lines:

- **HTTP status** stays the primary signal (`4xx` / `5xx`).
- **Body** includes fields the UI can rely on, for example: a machine-readable `code` (or `error`), a human-facing `message`, and **`request_id`** echoing the correlation header so support and logs line up with the browser.
- **422** vs **400**: pick one style for validation failures and document it; mixed behavior across endpoints confuses generated clients and TanStack Query error paths.

Keep internal stack traces and sensitive details out of the public body. The global error handler is the right place to enforce this consistently.

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

### Sharing the contract with a React client

Treat the OpenAPI document as the **single source of truth** both servers and clients compile from. Typical TypeScript workflows:

- generate **types** or **fetch clients** with **[openapi-typescript](https://github.com/drwpow/openapi-typescript)**, **[Hey API OpenAPI TS](https://github.com/hey-api/openapi-ts)**, or **[Orval](https://orval.dev/)**;
- consume those types in **TanStack Query** hooks and keep hand-written DTOs in the frontend to a minimum;
- fail CI when the spec drifts from implementation (contract tests or codegen in the pipeline — see the [DevOps playbook](https://github.com/khasky/devops-delivery-playbook-2026)).

### Monorepos and a shared contract package

In a **pnpm** (or npm) monorepo, a dedicated workspace package (for example `packages/api-contract`) can hold the OpenAPI file and published types. Useful habits:

- **`openapi.yaml`** lives in one place; API and `apps/web` both depend on the generated output;
- **Turborepo** / **Nx**: declare a `codegen` (or `openapi:generate`) task and make `build` depend on it so the UI never builds against stale types;
- version the package with the API when breaking changes ship so React code and Node services upgrade together.

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

**Default for new Node + TypeScript services:** **[Vitest](https://vitest.dev/)** for unit and HTTP integration tests (often with **Supertest** or the framework’s inject helper). **Jest** is still a reasonable choice when you are extending an existing Jest-heavy repo; align the runner with the [frontend](https://github.com/khasky/frontend-architecture-playbook-2026) and [DevOps](https://github.com/khasky/devops-delivery-playbook-2026) playbooks so CI commands stay obvious (`vitest run` vs `npm test -- --runInBand`).

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
  -> global pipeline attaches request_id
  -> auth step in pipeline resolves caller
  -> route-scoped validation validates body/query/path
  -> handler maps input DTO -> domain request
  -> service executes business use case
  -> repository loads/saves persistence models
  -> service returns domain result
  -> handler maps domain result -> response DTO
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

The HTTP entrypoint is thin. The service is meaningful. The response shape is explicit. That is the general goal.

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
    middleware/
    routes/
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
