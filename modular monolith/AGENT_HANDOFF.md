# Agent Handoff: Spring Boot Modular Monolith Claude Workflow

Give this file to a new agent first. It explains the goal, how we work, what is decided, and what is left.

## Goal

We are building a **reusable Claude workflow for Spring Boot modular monolith applications**: a set of written coding rules, later enforced by hooks and settings, that any Claude agent follows when building such an app.

Work folder: `modular monolith/`

## Read these first (fully, before doing anything)

1. `docs/coding-rules/backend.md`: the source of truth. Sections 1 to 13 are written.
2. `visual/progress.html`: living visual page. It shows only two things: the "Total structure so far" card and the one rule currently being designed.

## How we work (strict)

- Go step by step. The user says what to make. Do not create or change files the user did not ask for.
- For each topic: **explain or recommend first**, with one clear recommendation and the decisions the user must make. Then show the example (folders, code, JSON) in `visual/progress.html` as a "proposal". Only after the user confirms, write it into `backend.md` and mark it done on the page.
- When the user answers "go with your recommendations", apply them all.
- `visual/progress.html` holds only the Total structure card and the current rule. Update the total structure whenever the structure changes. Mark new items with `<b>new</b>`, and remove the marks when the section is confirmed.
- Hooks and `settings.json` changes come **last**, after everything else exists.
- No over-engineering. Use SOLID. Use patterns (Strategy, Adapter, Chain of Responsibility) only when a real need exists (rule of three).
- Short, direct answers. If asked "what do you recommend", give one recommendation.
- The user handles Keycloak and security. Docker files (Dockerfile, docker-compose) are a separate part after the backend.

## What is decided (details are in backend.md)

| # | Topic |
|---|---|
| 1 | Layout and exceptions: `shared/` plus modules. Each has `exception/{handler,types,message}`. One global handler (lowest precedence), one handler per module (highest precedence, only the module's base exception). |
| 2 | API layers: `api/{controller, dto/{request, response, mapper/response}, response/message, validation, docs}`, `model/`, `service/`, `entities/`. DTOs and entities never know each other, they meet only through the model via mappers. Naming `<Source>To<Target>Mapper`, placed in the target's folder. |
| 3 | Response envelope `ApiResponse<T>`. Proper HTTP codes. DELETE returns 200 with the envelope. 409 for state conflicts, 422 for business rule violations. |
| 4 | Logging: SLF4J fluent API with event names, JSON logs, own `CorrelationIdFilter` (`X-Correlation-Id`, MDC), no traceId/spanId for now. One owner per failure (the exception handlers). |
| 5 | API docs with springdoc, composed doc annotations in `api/docs`. Versioning `/api/v1` (major only). Plural kebab-case endpoints. |
| 6 | Messages: all text in `.properties` under `resources/messages/<module>/{api/response, api/validation, exception}/`. English only. Keys mirror the file path. Key constants in `message/` folders. Error code enums carry messageKey and HTTP status. |
| 7 | Validation: shape validation before the service (`api/validation`), business validation in `<module>/validation/validator`, database constraints as the last safety net. |
| 8 | Services: single responsibility, step-style public methods, no forwarding wrappers. Command service and query service split by intent. Public service interfaces vs internal services. Events via Spring `ApplicationEventPublisher` (events, payload, listener folders; listeners thin, async, after commit). Integrations behind ports and adapters (Stripe in payments, notifications module with channels, ai module with provider strategy, S3 in shared, Redis through Spring cache). |
| 9 | Entities: UUID v7 generated in the app, `BaseEntity` with audit fields and `@Version`, Lombok conventions, never `@Data`, IDs only across modules, Flyway with timestamp versions, `ddl-auto: validate`. |
| 10 | Lombok conventions for every class type. |
| 11 | Configuration: `application.yml` plus `-dev`, `-docker`, `-prod`. Secrets via env vars and `.env` / `.env.example`. Typed `@ConfigurationProperties` records. |
| 12 | Pagination, filtering and sorting: zero-based page, size 20 (max 100), one sort field with an allowlist per module, default `createdAt,desc`, shared `PageRequestDto` plus module `XSearchRequest`, Specifications only for optional combined filters, `PagedResponses.of(...)`, totals always counted. |
| 13 | Background jobs: thin `<module>/job/` triggers, schedule in yml (`fixedDelay` default), ShedLock on PostgreSQL, idempotent jobs, bounded batches with one transaction per batch, the next run is the retry, `@Retryable` only around vendor calls. |

## Status of the remaining backend topics

Order the user chose: pagination (done), caching, background jobs (done), idempotency and concurrency, testing, code style and build.

The user **skipped** these for now (nothing is written for them, the proposals below are only recommendations that were given and not confirmed). Ask the user if they want to return to them.

### Caching details (skipped)
Recommended: cache by id only, no list or search caching. Cache name `<module>.<thing>` plus a global `app:<env>:` prefix, names as constants. Default TTL 10 minutes with per-cache overrides in yml. Evict in the command service, with TTL as the safety net, transaction-aware cache manager, cross-module eviction only through events. JSON serialization of models only (never entities), version the cache name on breaking changes. Fail open with a WARN log if Redis is down. `sync = true` only on hot, expensive reads.

### Idempotency and concurrency (skipped)
Recommended: optimistic lock failure maps to 409 `CONCURRENT_MODIFICATION`, with `version` sent in the body on PUT and PATCH. Repeated DELETE returns 404. Idempotency keys for POST in PostgreSQL, opt-in `@Idempotent` annotation, `Idempotency-Key` header mandatory on those endpoints, 24-hour expiry cleaned by a job (409 `REQUEST_IN_PROGRESS`, 422 `IDEMPOTENCY_KEY_REUSED`). Listeners and webhooks stay safe through unique constraints on external ids, no inbox table yet. Pessimistic locking only for proven hot rows. Vendor calls with side effects send their own idempotency key to the vendor.

### Testing (the next topic, recommendations given, not yet confirmed)
Explain first, then wait for the user's decisions. The recommendations:

- **Pyramid:** many unit tests, a few slices, very few full integration tests.
- **Unit:** JUnit 5, Mockito, AssertJ, no Spring context. Services are tested with a **fake** implementation of the port. Mockito only for simple collaborators. Test names state the behavior (`placeOrder_whenStockMissing_throwsOutOfStock`). Arrange, Act, Assert.
- **Web slice:** `@WebMvcTest` with `@MockitoBean` services. Proves status codes, the `ApiResponse` envelope, correlation header, error codes (`VALIDATION_ERROR`, `INVALID_SORT`). Security is switched off or uses a fake JWT.
- **Data slice:** `@DataJpaTest` on **real PostgreSQL via Testcontainers, no H2**. One shared singleton container. Flyway runs for real, so a broken migration fails a test. Tests repositories, specifications and constraints.
- **Integration:** a few `@SpringBootTest` flows per module (create, fetch, cancel) with the real container. Third-party adapters replaced by fakes, no real Stripe or SMTP.
- **ArchUnit boundary tests** in `src/test/java/com/app/architecture/`, starting with seven rules:
  1. DTOs and entities do not depend on each other.
  2. Controllers do not touch entities or repositories.
  3. Services do not import DTOs, and not another module's entities.
  4. A module imports only another module's model, public service and event classes and payloads.
  5. No `@Data`, no field injection, no `@SneakyThrows`.
  6. Mapper names follow `<Source>To<Target>Mapper` and sit in the target's folder. No `*Impl` outside `impl/`.
  7. Vendor SDK packages (for example `com.stripe`) are imported only inside `integration/` adapters.
- **Layout:** test folders mirror main (`api/`, `service/`, `validation/`, `job/`, `repository/`), plus `shared/` test support (base classes, fakes, builders like `OrderModels.anOrder()`). Names: `<Class>Test` for unit and slice, `<Flow>IT` for integration.
- **Coverage:** JaCoCo, gate of about 80% on services and validators only (not DTOs, entities or config), defined in the build topic.
- **Not tested:** Lombok output, plain getters, records, configuration classes, the framework itself.
- Tests already required by earlier sections: custom constraint validators, request classes, each business validator rule, mappers, specifications, `PageRequestDtoToPageableMapper`, the job's service method (including a second run that does nothing), a fake port for services.

### Code style and build (not started)
Formatter and style rules, build tool setup, JaCoCo gate, ArchUnit in the build, and related checks.

## After the backend

1. Hooks and `settings.json` changes that enforce the rules (done last).
2. Docker files (Dockerfile, docker-compose), as a separate part.
3. The user will say what comes after that. Ask them.

## Current page state

`visual/progress.html` shows the total structure including everything through section 13. There is no open proposal at the moment. The next proposal replaces the current rule card, and the finished card is removed because its content lives in `backend.md`.
