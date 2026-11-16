# Backend Coding Rules (Spring Boot, Modular Monolith)

## 1. Project layout

- The app is split into multiple **modules**. Anything shared between modules (config, utils, common exceptions, etc.) lives in the top-level `shared` folder, outside every module.

## 2. Exception handling

Multiple `@RestControllerAdvice` classes are allowed: one global handler in `shared`, plus one handler per module.

### Folder structure

The same structure is used in `shared` and in every module:

```
shared/
  exception/
    handler/
      GlobalExceptionHandler.java
    types/
      <shared exception classes>
<module>/
  exception/
    handler/
      <Module>ExceptionHandler.java
    types/
      <module exception classes>
```

### Rules

1. `shared/exception/handler/GlobalExceptionHandler` is the fallback. It uses `@Order(Ordered.LOWEST_PRECEDENCE)` and handles generic cases (`Exception`, validation, not found, 500).
2. Each module owns `<Module>ExceptionHandler` in its own `exception/handler/` folder. It is scoped with `@RestControllerAdvice(basePackages = "<module package>")` and uses `@Order(Ordered.HIGHEST_PRECEDENCE)`.
3. A module handler must never catch `Exception` or `RuntimeException`. Doing so would shadow the global handler.
4. A module's exception classes live in its own `exception/types/` folder. Other modules must not import them.
5. Exceptions used by more than one module live in `shared/exception/types/`.
6. All handlers return the same `ErrorResponse` shape, defined in `shared`.
7. Java package names are lowercase with no underscores (`exception.handler`, `exception.types`).

## 3. API, model and entity layers

DTOs and entities must never know each other. Both only meet the internal **model**, and only through mappers.

### Folder structure (per module)

```
<module>/
├── api/
│   ├── controller/
│   │   └── XController.java
│   └── dto/
│       ├── request/    XRequest.java
│       ├── response/   XResponse.java
│       └── mapper/
│           └── response/   XModelToXResponseMapper.java        (Model -> Response)
├── model/
│   ├── XModel.java
│   └── mapper/
│       ├── CreateXRequestToXModelMapper.java                   (Request -> Model)
│       └── XEntityToXModelMapper.java                          (Entity -> Model)
├── service/        XService.java
├── entities/
│   ├── XEntity.java
│   └── mapper/     XModelToXEntityMapper.java                  (Model -> Entity)
├── enums/          enums used by entities, models and DTOs
└── constants/
    └── enums/      enums that are not tied to an entity (for example XErrorCode)
                    (message key constants live next to their use, see section 7)
```

`shared` follows the same shape where it applies (`exception`, `constants`, `enums`, `config`, `utils`).

### Rules

1. `XRequest` and `XResponse` never reference an entity. `XEntity` never references a DTO. DTOs and entities do not reference each other.
2. Request flow: request -> model (`model/mapper`), model -> entity (`entities/mapper`), entity -> model (`model/mapper`), model -> response (`api/dto/mapper/response`). Mappers are the only classes that know two sides.
3. Services work only with models. They never receive or return a DTO or an entity.
4. Controllers work only with DTOs and models. They never touch entities.
5. Other modules may use only another module's model, its public service and its event classes and payloads. They must not import its entities, DTOs, mappers, repositories or listeners.
6. Enums shared by entities, models and DTOs live in the module's `enums/` folder.
7. One mapper class per source type and direction. Mappers contain no business logic.
8. Mapper naming: `<Source>To<Target>Mapper` (for example `CreateOrderRequestToOrderModelMapper`, `OrderEntityToOrderModelMapper`, `OrderModelToOrderEntityMapper`, `OrderModelToOrderResponseMapper`). Each mapper lives in the folder of its target type: `model/mapper`, `entities/mapper` or `api/dto/mapper/response`. There is no `api/dto/mapper/request` folder.
9. MapStruct may be used to generate mappers. The folder rules stay the same.

## 4. Response format

Every API response uses one envelope, `ApiResponse<T>`, defined in `shared/response/`. This covers success, paginated success and error. Proper HTTP status codes are always sent.

### Objects (`shared/response/`)

```java
public record ApiResponse<T>(
    boolean success, String message,
    T data,                // null on error
    PageInfo pagination,   // only on paged responses
    ErrorInfo error,       // only on errors
    String correlationId,  // read from MDC
    Instant timestamp) {}

public record PageInfo(int page, int size, long totalElements,
                       int totalPages, boolean hasNext, boolean hasPrevious) {}
public record ErrorInfo(String code, List<FieldErrorInfo> details) {}
public record FieldErrorInfo(String field, String message) {}
```

Null fields are omitted from the JSON (`@JsonInclude(NON_NULL)`).

### HTTP status codes

| Status | Use |
|---|---|
| 200 OK | GET, PUT, PATCH, DELETE, paginated lists |
| 201 Created | POST that creates a resource |
| 400 Bad Request | validation errors, malformed JSON |
| 401 Unauthorized | not authenticated |
| 403 Forbidden | authenticated but not allowed |
| 404 Not Found | resource missing |
| 409 Conflict | state conflicts and duplicates (e.g. `ORDER_ALREADY_SHIPPED`) |
| 422 Unprocessable Entity | business rule violated |
| 500 Internal Server Error | anything unexpected, with no stack trace sent to the client |

### Rules

1. Controllers return `ResponseEntity<ApiResponse<T>>` and set the status explicitly.
2. DELETE returns `200` with the envelope, not `204`, so every response has the same shape.
3. The global and module exception handlers return `ApiResponse<Void>` with `error` filled in. They replace the separate `ErrorResponse` from section 2.
4. `error.code` is a stable string. Each module defines its codes in `<module>/constants/enums/<Module>ErrorCode`. Clients switch on the code, not the message.
5. Never return a stack trace or internal exception message in a 500 response.
6. Lists without paging return `data` as an array with no `pagination` field.
7. `correlationId` comes from the logging context (MDC), set by a shared `CorrelationIdFilter`. The filter reads the `X-Correlation-Id` request header, or generates a UUID if it is missing. It also returns the ID in the `X-Correlation-Id` response header and clears the MDC after the request.
8. Paging goes through a shared helper that turns a `Page<Model>` into `ApiResponse<List<Response>>` using the module's response mapper.

## 5. Logging

Source: `Production_Backend_Logging_Notes.pdf` (Drive, `logs` folder). Stack: SLF4J 2 with Logback, structured JSON output.

### Logger setup

- Each class that logs declares one logger: `@Slf4j` (Lombok) or `private static final Logger log = LoggerFactory.getLogger(X.class)`.
- Output is structured JSON on the console. Every line carries `@timestamp`, `level`, `service`, `environment`, `version`, `logger`, and `correlationId`.
- Production root level is `INFO`. `DEBUG` is enabled temporarily for one package at a time, with an owner and an expiry.

### Correlation ID

- A shared `CorrelationIdFilter` (in `shared`) sets `correlationId` in the MDC for every request.
- It reads the `X-Correlation-Id` request header. If missing, it generates a UUID. An incoming value is validated (length and allowed characters) and replaced when invalid, so it cannot be used for log injection.
- It returns the ID in the `X-Correlation-Id` response header and in the `ApiResponse.correlationId` field.
- It clears the MDC when the request ends.
- Async executors and outgoing HTTP calls must propagate the ID.
- Code never adds `correlationId` to a log call by hand. The logging framework attaches it from the MDC. This keeps a later move to tracing (`traceId` renamed to `correlationId`, plus `spanId`) a configuration change only.

### Structured events

Important logs use the SLF4J fluent API with a stable `event` name:

```java
log.atInfo()
   .addKeyValue("event", "order.create.completed")
   .addKeyValue("order_id", saved.getId())
   .addKeyValue("outcome", "success")
   .log("Order created");
```

1. `event` is the stable identifier for searching and automation. `message` is the readable text.
2. Event names follow `<module>.<action>.<outcome>`, for example `order.create.completed` and `payment.charge.failed`.
3. Field names are `snake_case` (`order_id`, `error_code`, `duration_ms`). Durations carry their unit.
4. Log only facts the code actually knows.

### Log levels

| Level | Use |
|---|---|
| ERROR | unexpected failure or exhausted recovery |
| WARN | degraded or expected rejected outcome that may recover |
| INFO | meaningful business outcome or lifecycle event |
| DEBUG | focused investigation detail |
| TRACE | extremely detailed steps, almost never |

- Validation failures and expected business errors are WARN or INFO without a stack trace, never ERROR.

### Where logging happens

1. Controllers do not log routine entry and exit.
2. Services log business outcomes and important decisions.
3. Repositories and mappers do not log.
4. Never write `Entering ...` or `Leaving ...` lines.
5. Do not log "created", "updated" or "committed" before the transaction has committed.
6. Background jobs log start (when useful), completion and terminal failure, with a stable `job_id` and an `attempt` field.

### One owner per failure

1. Each unexpected exception is logged once, with its stack trace, in the exception handlers (global and module).
2. Lower layers wrap with the original cause and rethrow. They never log and rethrow.
3. Expected business exceptions are logged by their handler at WARN, without `setCause`.
4. Background workers, scheduled tasks and message consumers are their own failure boundaries and log their own terminal failures.
5. Clients get the safe error envelope with `correlationId`. They never get a stack trace.

### Cost control

1. Use `{}` placeholders. Never concatenate strings.
2. Guard expensive arguments with `isDebugEnabled()` or an SLF4J supplier (`addArgument(() -> ...)`).
3. Do not log whole DTOs or entities. Log identifiers, counts and outcomes.
4. Do not log every SQL row, loop item or stream token. Summarize batches instead.
5. Use bounded values for metric labels. Identifiers such as order IDs stay in logs only.

### Sensitive data

1. Never log passwords, tokens, cookies, authorization headers, payment details or full request and response bodies.
2. Log an allowlist of safe fields. Check exception messages and nested strings too.
3. Sanitize untrusted values to prevent log injection.
4. Records that must never be lost (audit trail) do not rely on the log pipeline. Use a database or outbox record saved in the same transaction.

## 6. API documentation, versioning and endpoint naming

API docs use OpenAPI through springdoc. Operation-level docs live in `api/docs/` as composed annotations, so controllers stay clean.

### Folder structure

```
shared/
├── constants/ApiPaths.java                  (V1 = "/api/v1")
└── docs/
    ├── annotations/StandardErrorsDoc.java   (400, 401, 403, 500 with examples)
    └── examples/ErrorExamples.java

<module>/api/
├── controller/XController.java
├── dto/
│   ├── request/    XRequest.java            (class + validation annotations)
│   └── response/   XResponse.java           (record)
└── docs/
    ├── annotations/    CreateXDoc.java, GetXDoc.java, ListXDoc.java ...
    └── examples/       XExamples.java       (JSON example strings as constants)
```

### Documentation rules

1. Request DTOs are classes, because they carry validation annotations. Response DTOs are records.
2. Each endpoint has one composed annotation in `api/docs/annotations/` named `<Action><Resource>Doc`. It holds the `@Operation` summary and description, and the `@ApiResponse` entries for success and each expected error.
3. Every documented response includes a JSON example in the full `ApiResponse` envelope shape. Examples are constants in `api/docs/examples/`, defined with text blocks.
4. Controllers carry only the composed annotations, for example `@CreateOrderDoc @StandardErrorsDoc`, plus `@Tag` on the class. No inline `@Operation` or `@ApiResponse` in controllers.
5. Errors common to every endpoint (400, 401, 403, 500) come from `@StandardErrorsDoc` in `shared/docs`. Module docs add only their own errors.
6. Field-level docs stay on the DTO fields as a short `@Schema` with a description and an example. They also state the expected value rules (`minimum`, `maxLength`, `allowableValues`) matching the validation annotations.
7. The swagger `@ApiResponse` name clashes with our `ApiResponse<T>`. In docs annotations the swagger one is written fully qualified.

### Versioning

1. The version goes in the path as a major number only: `/api/v1/...`. Never `1.0`.
2. Increase the version only for breaking changes.
3. The base path is the constant `ApiPaths.V1`, never a repeated string.

### Endpoint naming

1. Resources are plural nouns in lowercase kebab-case: `/api/v1/orders`, `/api/v1/order-items`.
2. No verbs in paths. The HTTP method is the verb.
3. Standard CRUD:

| Method | Path | Result |
|---|---|---|
| GET | `/api/v1/orders` | list (paged, filtered, sorted) |
| GET | `/api/v1/orders/{id}` | one |
| POST | `/api/v1/orders` | create, 201 |
| PUT | `/api/v1/orders/{id}` | full replace |
| PATCH | `/api/v1/orders/{id}` | partial update |
| DELETE | `/api/v1/orders/{id}` | delete, 200 with envelope |

4. Sub-resources: `/api/v1/orders/{id}/items`.
5. Non-CRUD actions are a POST with the action as the last segment: `POST /api/v1/orders/{id}/cancel`.
6. Query parameters and JSON fields use camelCase: `?page=0&size=20&sort=createdAt,desc&status=SHIPPED`.
7. No trailing slash, no file extension, and no module name in the URL.


## 7. Messages and validation

English only for now. No message text is hardcoded anywhere. Code holds message keys, `.properties` files hold the text. Other application config stays in `application.yml`.

### Validation levels

| Level | What it checks | Where | Result |
|---|---|---|---|
| Shape | required, length, range, format | Bean Validation annotations on the request class, triggered with `@Valid`, plus custom constraints in `api/validation/` | 400 `VALIDATION_ERROR` with a field list |
| Business | state and rules ("already shipped", "stock available") | `validation/validator/`, called by the service, throwing a module exception | 409 or 422 with a module error code |
| Database | unique and not-null constraints | the DB, as the last safety net | `DataIntegrityViolationException` is mapped to 409 |

### Folder structure

```
<module>/
├── api/
│   ├── response/
│   │   └── message/        XResponseMessages.java    (success message keys only)
│   └── validation/                                   (everything before the service)
│       ├── annotation/     custom constraint annotations
│       ├── validator/      ConstraintValidator classes
│       └── message/        CreateXRequestMessages.java
├── validation/                                       (business rules, called by the service)
│   └── validator/          XCancelValidator.java
├── constants/enums/        XErrorCode.java           (code + messageKey + HTTP status)
└── exception/
    ├── handler/            XExceptionHandler.java
    ├── types/              XException.java
    └── message/            XExceptionMessages.java   (key constants for all error text)

shared/
├── message/                MessageService.java, MessageConfig.java
├── validation/             constraints used by two or more modules
│   ├── annotation/  validator/  message/
└── exception/
    ├── types/              ErrorCode.java, BaseException.java
    └── message/            SharedExceptionMessages.java

src/main/resources/
├── application.yml
└── messages/
    ├── shared/
    │   ├── api/
    │   │   ├── response/response.properties
    │   │   └── validation/validation.properties
    │   └── exception/exception.properties
    └── <module>/
        ├── api/
        │   ├── response/response.properties          (success messages)
        │   └── validation/validation.properties      (request-shape messages)
        └── exception/exception.properties            (all error text)
```

`api/dto/response/` and `api/response/message/` are different. The first holds the response data record returned to the client. The second holds success message key constants only, with no data. Error text never lives under `api/response/`. It always comes from exceptions.

### Message rules

1. No message text in Java code, annotations, exceptions or controllers. Only keys.
2. Text lives in `src/main/resources/messages/` as `.properties` files, one file per area per module: `response.properties`, `validation.properties`, `exception.properties`.
3. Keys mirror the file path, so each key is globally unique and tells you where its text lives:
   - `messages/orders/api/response/` gives `order.api.response.created`
   - `messages/orders/api/validation/` gives `order.api.validation.amount.positive`
   - `messages/orders/exception/` gives `order.exception.already-shipped`
   - `messages/shared/exception/` gives `shared.exception.internal-error`
4. Keys are written as `public static final String` constants in key classes, one class per concern, with a private constructor. They sit next to the code that uses them: `api/response/message/`, `api/validation/message/` and `exception/message/`.
5. Annotations use keys in braces built from the constant: `@Positive(message = "{" + CreateOrderRequestMessages.AMOUNT_POSITIVE + "}")`.
6. `shared/message/MessageConfig` scans `classpath*:messages/**/*.properties` and registers every file as a bundle, so adding a module needs no config edit. It also wires Bean Validation (`LocalValidatorFactoryBean.setValidationMessageSource`) to the same bundles.
7. `shared/message/MessageService.get(key, args...)` resolves text using the request locale. Controllers and handlers use it. Nothing else resolves messages.
8. Placeholders such as `{0}` are used for dynamic values. Never concatenate text in code.
9. All error text comes from exceptions. A business rule failure, not found, malformed request and unexpected error each have their text in the `exception/` bundle (the module's, or `shared` for generic ones such as `validation-failed`, `malformed-request` and `internal-error`).

### Error codes and exceptions

1. Each module has an error code enum in `constants/enums/` implementing `ErrorCode` (`code()`, `messageKey()`, `status()`). Each entry carries its message key (from `XExceptionMessages`) and HTTP status.
2. `BaseException` holds an `ErrorCode` and arguments. It never holds message text.
3. Each module has one base exception (`OrderException`) that takes its own error code enum. The module handler has one `@ExceptionHandler` for that base type. It reads the status and key from the enum, resolves the text with `MessageService`, and returns `ApiResponse.error(...)`.
4. Module handlers never catch `Exception` or `RuntimeException`.
5. Business validators only throw. They never build responses or resolve messages.
6. `GlobalExceptionHandler` takes its generic texts from `SharedExceptionMessages`.

### Shape validation rules

1. Validation annotations go on request classes only. Not on models, entities or response records.
2. Use separate request classes for create and update. No validation groups. For PATCH, fields are nullable and validated only when present.
3. Keep annotation values in step with the field's `@Schema` (`maxLength`, `minimum` and so on).
4. Path and query parameters are validated with `@Validated` on the controller. The page size is capped in one shared place (default 20, max 100).
5. Custom constraints are an annotation plus a `ConstraintValidator`, in `api/validation/annotation/` and `api/validation/validator/`. Constraints used by more than one module go in `shared/validation/`. They are pure checks: no DB calls, no logging.
6. `GlobalExceptionHandler` handles every shape failure and returns 400:
   - `MethodArgumentNotValidException`: `VALIDATION_ERROR` with field details
   - `ConstraintViolationException`: `VALIDATION_ERROR` with field details
   - `HttpMessageNotReadableException`: `MALFORMED_REQUEST`
   - missing parameters and type mismatches: `VALIDATION_ERROR`
7. Rejected values are never echoed back to the client. Failures are logged at WARN with no stack trace and no submitted values.

### Business validation rules

1. A business validator is named `<Resource><Action>Validator`, one per use case, in `validation/validator/`.
2. The service calls it before doing the work. Validators take a model, never a DTO or an entity.
3. A validator may use repository queries that return only a boolean or a count (`existsByEmail`, `countByCustomerId`), or another module's public service. Never another module's repository or entities.
4. Reusable single rules may go in `validation/rules/`. Add that folder only when the same rule is shared by several validators.
5. Validators do not write to the database.
6. A validator has no message text. It throws the module exception with an error code, and the text comes from `exception/`.

### Tests

1. Each custom constraint validator has a unit test.
2. Each request class has a test for its main constraints.
3. Each business validator has a unit test for each rule.

## 8. Services, events, integrations, SOLID

### Service style

1. One responsibility per class. A class has one reason to change.
2. A public service method is the use case, written as a list of named steps:
   ```java
   @Transactional
   public OrderModel place(OrderModel order) {
       placeValidator.validate(order);
       PaymentModel payment = paymentService.charge(paymentRequestMapper.toRequest(order));
       OrderModel saved = save(order, payment);
       eventPublisher.publishEvent(new OrderPlacedEvent(OrderPlacedPayload.from(saved)));
       return saved;
   }
   ```
3. A private method holds logic that belongs to this class only, and names one step (for example `save(...)`, the service's own persistence).
4. A method that only forwards a call to another class is not written. Call that class directly.
5. If another class would also need the logic, it gets its own class: mapping goes to a mapper, payload creation to a `from(...)` factory on the payload.
6. A one-line step does not need its own method. A step that grows, or needs its own dependencies, becomes its own class.
7. No over-engineering: no interface, factory or pattern without a real need.
8. Services work only with models (see section 3), use constructor injection (`@RequiredArgsConstructor`, `final` fields), and use verb names for use cases: `place`, `cancel`, `getById`, `search`.
9. `@Transactional` goes on service methods only. Read-only methods use `@Transactional(readOnly = true)`. Transactions stay short: no remote HTTP calls or slow work inside one.
10. A service throws the module exception with an error code. It never returns null or a boolean to signal failure, and never catches and logs only to rethrow.

### Command and query services

1. Split services by intent, not by technical operation.
2. `XService` holds commands: `place`, `cancel`, `update`. It changes state.
3. `XQueryService` holds queries: `getById`, `search`. It is `@Transactional(readOnly = true)` and is where Redis caching (`@Cacheable`) is used. It returns `Page<XModel>` or `List<XModel>`, and takes a plain `Pageable` and a filter model.
4. A flow that reads and then writes is a command and lives in `XService`.
5. A command loads the data it needs from the repository directly. It never calls `XQueryService`, because cached results can be stale and a write needs fresh data inside its own transaction.
6. A small module with only one or two reads may use a single service. A large module splits from the start.

### Public and internal services

1. A module exposes only: its public service interfaces, its models, and its event classes and payloads.
2. A service used by other modules is an interface (the public API) plus an `impl/` class. A service used only inside the module is a plain class with no interface, and is package-private or in `service/internal/`.
3. Other modules call only public services, never repositories, entities, mappers, validators or internal services.

### Events

Events use Spring's built-in `ApplicationEventPublisher`. It runs in memory inside the application. No extra library is used.

```
<module>/
└── event/
    ├── OrderPlacedEvent.java               events this module publishes
    ├── payload/OrderPlacedPayload.java     the data each event carries
    └── listener/                           reactions to other modules' events
        └── PaymentCompletedOrderListener.java
```

1. A module's `event/` folder holds its own events, its `payload/`, and its `listener/` (reactions to other modules' events). Any of them may be absent.
2. The publisher owns the event and the payload. The receiving module owns its listener.
3. A reacting module imports only the publisher's event classes and payloads (`event/` and `event/payload/`), never its listeners or anything else.
4. Dependencies run one way, from the reacting module to the publishing module. Avoid loops.
5. There is no `publisher/` class. The publishing service calls `eventPublisher.publishEvent(...)` directly, and the payload has a static `from(model)` factory.
6. Event names are `<Thing><PastTense>Event`, for example `OrderPlacedEvent`.
7. Payloads are immutable records with IDs and small fields, never entities. They are stable public API: only add fields. A breaking change is a new versioned event.
8. A listener is thin. It turns an event into one call to a public service and contains no business logic.
9. Listeners use `@Async` with `@TransactionalEventListener(phase = AFTER_COMMIT)`. A failed listener is logged and retried, and never fails the original request.
10. `shared/config` enables async and provides the thread pool. The pool carries the correlation ID into the new thread (for example with `ContextPropagatingTaskDecorator`), so listener logs keep it.
11. Events exist only in memory and are lost if the app stops before a listener runs. If events must survive a crash, or a module becomes its own service, revisit with Spring Modulith's event publication registry or Spring Cloud Stream. Thin listeners and stable payloads keep that move cheap.

### Integrations (third-party libraries)

Third-party libraries sit behind our own interfaces in an `integration/` folder. Services depend on the interface, never on a vendor SDK.

```
payments/integration/
├── PaymentGateway.java                 port: our terms, our models
└── stripe/                             StripePaymentGateway, StripeProperties, StripeMapper

notifications/                          its own module: several kinds of messages
├── service/                            NotificationService (public), TemplateService (internal)
├── event/listener/                     reacts to other modules' events
├── integration/
│   ├── NotificationChannel.java        port
│   ├── NotificationChannelResolver.java
│   ├── email/                          EmailChannel (Google SMTP via JavaMailSender)
│   └── sms/  push/                     future
└── resources/templates/notifications/  one template per message type

ai/                                     its own module
├── service/                            AiService (public), PromptService, AiUsageService (internal)
└── integration/                        AiClient (port), AiClientResolver, anthropic/, openai/

shared/integration/storage/             FileStorage (port), s3/S3FileStorage (used by many modules)
shared/config/CacheConfig               Redis. Spring's cache is already the abstraction: no wrapper class
```

1. A capability owned by one module lives in that module's `integration/` folder. A capability used by two or more modules lives in `shared/integration/`.
2. Ports and adapters are named for the capability and the vendor: `PaymentGateway` and `StripePaymentGateway`, `FileStorage` and `S3FileStorage`.
3. Only the adapter imports the vendor SDK. Vendor types never appear in services, models, DTOs or exceptions.
4. The adapter maps vendor types to our models, and turns vendor exceptions into the module exception with an error code and the original cause attached. `BaseException` accepts a cause.
5. The adapter does not log. The exception handler logs the failure once.
6. Timeouts, retries and circuit breakers are set in the adapter and in `application.yml`. Secrets come from environment variables, never from code or committed files.
7. Settings are read with `@ConfigurationProperties` classes, one per vendor.
8. Notification text follows the message rules: templates in `resources/templates/notifications/`, fixed phrases as message keys.
9. Services are tested with a fake implementation of the port.

### SOLID and patterns

1. S: one job per class. A validator validates, a mapper maps, an adapter talks to one vendor.
2. O: a new provider or channel is a new class. No change to the service.
3. L: every adapter behaves like its port: same inputs, outputs and exceptions.
4. I: ports are small (`PaymentGateway`, `FileStorage`), not one large interface.
5. D: services depend on ports and public services. Spring injects the adapter.
6. Adapter: used for every third-party library.
7. Strategy: only when two or more real implementations exist at the same time (AI providers, notification channels). Spring injects `List<Port>`, and a resolver picks by key.
8. Chain of Responsibility: only for ordered, independent steps that can each pass or stop (for example fraud checks). Spring injects the ordered list.
9. Interface only at a boundary: a third-party library, two or more implementations, or another module's public service. Everything else is a plain class.
10. Rule of three: do not add a pattern until a real second case appears.

## 9. Entities

### Where things live

```
shared/
├── entities/
│   ├── BaseEntity.java                 id, createdAt, updatedAt, createdBy, updatedBy, version
│   └── BaseSoftDeleteEntity.java       opt-in: adds deletedAt
├── config/JpaAuditingConfig.java       @EnableJpaAuditing + AuditorAware
└── utils/UuidV7.java                   time-ordered UUID generator

<module>/
├── entities/
│   ├── XEntity.java
│   └── mapper/XModelToXEntityMapper.java   the only place that calls XEntity.builder()
└── repository/XRepository.java             returns entities, used only inside the module

src/main/resources/db/migration/
├── shared/
└── <module>/V<yyyyMMddHHmm>__<module>_<description>.sql
```

### BaseEntity

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter
@SuperBuilder
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public abstract class BaseEntity {

    @Id
    @Column(name = "id", nullable = false, updatable = false)
    @Builder.Default
    private UUID id = UuidV7.generate();

    @CreatedDate     @Column(name = "created_at", nullable = false, updatable = false) private Instant createdAt;
    @LastModifiedDate @Column(name = "updated_at", nullable = false)                   private Instant updatedAt;
    @CreatedBy       @Column(name = "created_by", updatable = false)                   private UUID createdBy;
    @LastModifiedBy  @Column(name = "updated_by")                                      private UUID updatedBy;

    @Version
    @Column(name = "version", nullable = false)
    private Long version;      // wrapper type: null means "new" to Spring Data

    // equals: same class family and same non-null id. hashCode: Objects.hashCode(id)
}
```

### An entity

```java
@Entity
@Table(name = "orders", indexes = @Index(name = "idx_orders_customer_id", columnList = "customer_id"))
@Getter @Setter
@SuperBuilder
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OrderEntity extends BaseEntity {

    @Column(name = "customer_id", nullable = false)
    private UUID customerId;          // another module's data: ID only

    @Column(name = "amount", nullable = false, precision = 12, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    @Builder.Default
    private OrderStatus status = OrderStatus.CREATED;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<OrderItemEntity> items = new ArrayList<>();
}
```

### Rules

1. **Lombok on entities:** `@Getter` and `@Setter` per class, never `@Data`. `@SuperBuilder` on `BaseEntity` and every entity (plain `@Builder` cannot set inherited fields). `@Builder.Default` for every field that has a default value. `@NoArgsConstructor(access = PROTECTED)` for JPA. No public all-args constructor. No `@ToString` and no `@EqualsAndHashCode` on entities.
2. **Builders only in the entity mapper.** The mapper sets `id` only when the model already has one (an existing record). For a new record it leaves `id` unset, so the `@Builder.Default` generates it. Passing `id(null)` would override the default.
3. **Id:** UUID v7, created in the application with `UuidV7.generate()`, so it exists before saving. Never random v4 ids on large tables.
4. **Audit:** `createdAt` and `updatedAt` are `Instant` in UTC, and `createdBy` and `updatedBy` are UUIDs, all filled by Spring Data auditing. Every entity extends `BaseEntity`.
5. **Locking:** every entity has a `@Version Long` field (wrapper type) for optimistic locking.
6. **equals and hashCode** use `id` only. `toString` never includes relations.
7. **Relations:** `LAZY`, always set explicitly (`@ManyToOne` is eager by default). Cascade `ALL` and `orphanRemoval` only for children that cannot exist without the parent. Bidirectional links only when a query needs them.
8. **Module boundaries:** another module's data is referenced by ID only. No JPA relation and no cross-module foreign key.
9. **Columns:** snake_case column names and plural snake_case table names. Indexes and constraints have explicit names. `@Enumerated(EnumType.STRING)`, never ordinal. `BigDecimal` with explicit `precision` and `scale` for money, never `double`.
10. **Entities hold persistence only:** no business logic and no Bean Validation annotations (those are on requests). Database rules go in `@Column` (`nullable`, `length`) and in migrations.
11. **Deletes:** real deletes by default. Soft delete only on entities that extend `BaseSoftDeleteEntity` (`deletedAt`, with `@SQLDelete` and `@SQLRestriction("deleted_at IS NULL")`).
12. **Schema:** Flyway only, with `ddl-auto: validate`. One folder per module under `db/migration/`. Migration versions are timestamps (`V202611161030__orders_create_orders.sql`), so they never collide across modules.
13. **Repositories** are in `<module>/repository/`, return entities, and are used only inside the module (services and validators). They are never exposed to other modules. Avoid N+1 queries with a fetch join or a projection.
14. **Time:** store and handle everything in UTC (`hibernate.jdbc.time_zone: UTC`).
15. **application.yml:** `spring.jpa.open-in-view: false` and `spring.jpa.hibernate.ddl-auto: validate`.

## 10. Lombok conventions (all classes)

| Class type | Annotations |
|---|---|
| Service, validator, mapper, listener, handler, adapter, any Spring bean with dependencies | `@RequiredArgsConstructor` with `final` fields, plus `@Slf4j` if it logs |
| Entity | see section 9 |
| Request DTO (class) | `@Getter @Setter @NoArgsConstructor`. Add `@Builder` and `@AllArgsConstructor` only when tests or mappers need them. |
| Response DTO, model, event, payload | Java records, no Lombok. `@With` is allowed on a record for copy-with-change. |
| Exception | `@Getter` only, where it holds fields |
| `@ConfigurationProperties` | a record, or `@Getter @Setter` on a class |
| Constants and message key classes | `final class` with `@NoArgsConstructor(access = AccessLevel.PRIVATE)` |

Never use:
1. `@Data`
2. `@Value` on entities
3. `@SneakyThrows`
4. field injection (`@Autowired` on fields)
5. a public `@AllArgsConstructor` on an entity

## 11. Configuration and profiles

Keycloak and security rules are handled separately. Only the Keycloak `issuer-uri` setting appears here. Docker files (`Dockerfile`, `docker-compose.yml`) get their own part later.

### Files

```
src/main/resources/
├── application.yml            shared by every profile
├── application-dev.yml        IDE: localhost, DEBUG, readable logs
├── application-docker.yml     local Docker: compose hostnames, JSON logs
└── application-prod.yml       production: ${VAR} placeholders only, no secret defaults

project root/
├── .env                       git-ignored, real values for the IDE
└── .env.example               committed, every variable with a dummy value
```

| File | Used when | Holds |
|---|---|---|
| `application.yml` | always | app name, `spring.profiles.default: dev`, Jackson, JPA (`ddl-auto: validate`, `open-in-view: false`, UTC), Flyway locations, graceful shutdown, `include-stacktrace: never`, base log level and structured log fields, `management` basics, defaults for `app.*`. No hosts and no secrets. |
| `application-dev.yml` | running from the IDE | `localhost` for the DB, Redis and Keycloak, `DEBUG` for `com.app`, a readable console pattern with `correlationId` and `%kvp`, Swagger on, Stripe test key from `.env` |
| `application-docker.yml` | local Docker | compose hostnames (`postgres`, `redis`), values from environment variables, JSON logs (`logging.structured.format.console: logstash`), Swagger on |
| `application-prod.yml` | production | `${VAR}` placeholders only for hosts and secrets, JSON logs, Swagger off, actuator limited to `health` and `prometheus`, tuned connection pool |

### Rules

1. Profile names are `dev`, `docker` and `prod`. The default profile is `dev` (`spring.profiles.default: dev`), so the app runs from the IDE with no flags. Docker sets `SPRING_PROFILES_ACTIVE=docker` and production sets `prod`.
2. Only differences go in a profile file. Anything identical everywhere stays in `application.yml`.
3. Secrets are never committed. Profile files hold `${VAR}` placeholders. IDE values come from a git-ignored `.env`. `.env.example` lists every variable with a dummy value.
4. In `prod`, hosts and secrets have no default value, so a missing variable stops the app at startup. `dev` may use defaults for local-only values.
5. Log format: `dev` uses a readable console pattern that shows `correlationId`. `docker` and `prod` use JSON (`logstash` format), so Docker behaves like production.
6. Our own settings live under `app.<module>.*` (for example `app.payments.stripe.timeout`) and are read through `@ConfigurationProperties` records with `@Validated`, so a missing or invalid value fails at startup. The application class has `@ConfigurationPropertiesScan`.
7. Property keys are kebab-case.
8. Swagger is on in `dev` and `docker`, and off in `prod`.
9. If `application.yml` passes about 300 lines, split it by module with `spring.config.import`.

### Typed settings example

```java
// payments/integration/stripe/StripeProperties.java
@Validated
@ConfigurationProperties(prefix = "app.payments.stripe")
public record StripeProperties(
        @NotBlank String secretKey,
        @NotNull Duration timeout) {}
```

## 12. Pagination, filtering and sorting

### Where things live

```
shared/
├── constants/PageDefaults.java                  DEFAULT_SIZE=20, MAX_SIZE=100, DEFAULT_SORT="createdAt,desc"
├── pagination/
│   ├── PageRequestDto.java                      page, size, sort (validated class)
│   └── PageRequestDtoToPageableMapper.java      dto + allowed fields -> Pageable
├── response/PagedResponses.java                 Page<Model> -> ApiResponse<List<Response>> with PageInfo
├── exception/types/InvalidSortException.java    400 INVALID_SORT, handled by GlobalExceptionHandler
└── docs/annotations/PageableDoc.java            documents page, size and sort once

<module>/
├── api/dto/request/XSearchRequest.java          filter fields (validated class)
├── model/XSearchModel.java                      record with plain filter values
├── model/mapper/XSearchRequestToXSearchModelMapper.java
├── constants/XSortFields.java                   allowlist of sortable fields
└── repository/
    ├── XRepository.java                         extends JpaRepository and JpaSpecificationExecutor
    └── specification/XSpecifications.java       only for optional, combined filters
```

### Request flow

`GET /api/v1/orders?page=0&size=20&sort=createdAt,desc&status=SHIPPED`

1. `PageRequestDto` and `XSearchRequest` are validated (400 if bad).
2. `PageRequestDtoToPageableMapper` checks the sort field against `XSortFields` and builds the `Pageable`.
3. The search request mapper builds the `XSearchModel`.
4. `XQueryService.search(pageable, filter)` returns `Page<XModel>`.
5. `PagedResponses.of(page, responseMapper::map, message)` builds `ApiResponse<List<XResponse>>` with `pagination`.

### Shared request and mapper

```java
@Getter @Setter @NoArgsConstructor
public class PageRequestDto {

    @Min(value = 0, message = "{" + PageRequestMessages.PAGE_MIN + "}")
    private int page = 0;

    @Min(value = 1,                     message = "{" + PageRequestMessages.SIZE_RANGE + "}")
    @Max(value = PageDefaults.MAX_SIZE, message = "{" + PageRequestMessages.SIZE_RANGE + "}")
    private int size = PageDefaults.DEFAULT_SIZE;

    // one field only: "field" or "field,asc|desc". A second sort parameter fails the pattern.
    @Pattern(regexp = "^[A-Za-z]+(,(asc|desc))?$", message = "{" + PageRequestMessages.SORT_FORMAT + "}")
    private String sort = PageDefaults.DEFAULT_SORT;
}

@Component
public class PageRequestDtoToPageableMapper {

    public Pageable map(PageRequestDto dto, Set<String> allowedSortFields) {
        String[] parts = dto.getSort().split(",");
        String field = parts[0];
        if (!allowedSortFields.contains(field)) {
            throw new InvalidSortException(field, allowedSortFields);   // key shared.exception.invalid-sort
        }
        Sort.Direction direction = parts.length > 1 ? Sort.Direction.fromString(parts[1]) : Sort.Direction.ASC;
        return PageRequest.of(dto.getPage(), dto.getSize(), Sort.by(direction, field));
    }
}
```

### Module pieces

```java
@Getter @Setter @NoArgsConstructor
public class OrderSearchRequest {
    private OrderStatus status;      // all filters optional
    private UUID customerId;
    private Instant createdFrom;
    private Instant createdTo;
}

public record OrderSearchModel(OrderStatus status, UUID customerId, Instant createdFrom, Instant createdTo) {}

@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class OrderSortFields {
    public static final Set<String> ALLOWED = Set.of("createdAt", "amount", "status");   // entity field names
}

@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class OrderSpecifications {
    public static Specification<OrderEntity> from(OrderSearchModel f) {
        return Specification.allOf(hasStatus(f.status()), forCustomer(f.customerId()),
                                   createdFrom(f.createdFrom()), createdTo(f.createdTo()));   // null parts are ignored
    }
    private static Specification<OrderEntity> hasStatus(OrderStatus s) {
        return s == null ? null : (root, q, cb) -> cb.equal(root.get("status"), s);
    }
}
```

### Query service, controller, helper

```java
// service
@Transactional(readOnly = true)
public Page<OrderModel> search(Pageable pageable, OrderSearchModel filter) {
    return orderRepository.findAll(OrderSpecifications.from(filter), pageable)
                          .map(orderEntityToOrderModelMapper::map);
}

// controller
@GetMapping
@ListOrdersDoc @PageableDoc @StandardErrorsDoc
public ResponseEntity<ApiResponse<List<OrderResponse>>> search(
        @Valid @ParameterObject PageRequestDto paging,
        @Valid @ParameterObject OrderSearchRequest filter) {
    Pageable pageable = pageableMapper.map(paging, OrderSortFields.ALLOWED);
    Page<OrderModel> page = orderQueryService.search(pageable, searchMapper.map(filter));
    return ResponseEntity.ok(PagedResponses.of(page, orderResponseMapper::map,
            messageService.get(OrderResponseMessages.LISTED)));
}

// shared helper
public static <M, R> ApiResponse<List<R>> of(Page<M> page, Function<M, R> mapper, String message) {
    PageInfo info = new PageInfo(page.getNumber(), page.getSize(), page.getTotalElements(),
                                 page.getTotalPages(), page.hasNext(), page.hasPrevious());
    return ApiResponse.paged(message, page.map(mapper).getContent(), info);
}
```

### Responses

```json
{
  "success": true,
  "message": "Orders retrieved",
  "data": [ { "id": "0192...", "status": "SHIPPED", "amount": 49.90 } ],
  "pagination": { "page": 0, "size": 20, "totalElements": 57, "totalPages": 3, "hasNext": true, "hasPrevious": false },
  "correlationId": "7c1e...",
  "timestamp": "2026-11-16T10:30:00Z"
}
```

- A page past the end returns `"data": []` with 200.
- `?sort=password` returns 400 with `error.code` `INVALID_SORT` and `details: [{ "field": "sort", "message": "Allowed: amount, createdAt, status" }]`.
- `?size=500` returns 400 `VALIDATION_ERROR` with field `size`.

### Rules

1. Parameters: `page` (zero-based, default 0), `size` (default 20, max 100), `sort` (`field` or `field,asc|desc`, one field only).
2. All defaults live in `shared/constants/PageDefaults`. The default sort is `createdAt,desc`. A module may override its default.
3. A list endpoint takes the shared `PageRequestDto` and a module `XSearchRequest` as two separate parameters. Neither extends the other. Both are validated classes.
4. Each module has `XSortFields`, an allowlist whose names match entity fields. Any other field is a 400 `INVALID_SORT`.
5. The query service takes a `Pageable` and a filter model and returns `Page<XModel>`. No DTO reaches the service.
6. Use Specifications in `repository/specification/` only for optional, combined filters. A fixed filter uses a plain repository method (`findByCustomerId`).
7. Controllers never build `PageInfo` by hand. They use `PagedResponses.of(...)`.
8. Totals are always counted. Switch a heavy table to `Slice` only when a real need appears.
9. A page past the end returns an empty list with 200.
10. `@PageableDoc` documents `page`, `size` and `sort` once. Each module's doc annotation adds only its own filter fields.
11. Message keys: `messages/shared/api/validation/` for page, size and sort, and `messages/shared/exception/` for `invalid-sort`.
12. Tests: `PageRequestDto` constraints, `PageRequestDtoToPageableMapper` (allowed, rejected, default), and each specification.

## 13. Background jobs

### Where things live

```
shared/
└── config/SchedulingConfig.java        @EnableScheduling, @EnableSchedulerLock, @EnableRetry, job thread pool, LockProvider

<module>/
├── job/
│   ├── ExpiredOrderCleanupJob.java     thin trigger: lock, call service, log
│   └── OrdersJobProperties.java        @ConfigurationProperties record: batch size
└── service/OrderService.java           expireBatch(batchSize): one batch, one transaction

src/main/resources/db/migration/shared/
└── V202611161100__shared_create_shedlock.sql
```

### Flow of one run

1. The scheduler thread (dedicated pool) sets a new `correlationId` in the MDC.
2. `@SchedulerLock` lets one instance run. Other instances skip silently.
3. The job logs its start (when useful), then loops over `orderService.expireBatch(batchSize)`. Each call is one short transaction and returns the number of rows processed.
4. The loop stops when a batch returns less than `batchSize`, or when the max-batches safety stop is reached.
5. The job logs `completed` with `processed_count` and `duration_ms`. On an unexpected failure it logs `failed` once with the stack trace.
6. The next `fixedDelay` run picks up whatever is left. That is the retry.

### The job

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class ExpiredOrderCleanupJob {

    private static final String JOB_ID = "orders.expired-cleanup";
    private static final int MAX_BATCHES = 100;                       // safety stop

    private final OrderService orderService;
    private final OrdersJobProperties properties;

    @Scheduled(fixedDelayString = "${app.orders.jobs.expired-cleanup.fixed-delay}")
    @SchedulerLock(name = JOB_ID,
            lockAtMostFor  = "${app.orders.jobs.expired-cleanup.lock-at-most-for}",
            lockAtLeastFor = "${app.orders.jobs.expired-cleanup.lock-at-least-for}")
    public void run() {
        long start = System.nanoTime();
        int total = 0;
        try {
            int batchSize = properties.expiredCleanup().batchSize();
            for (int i = 0; i < MAX_BATCHES; i++) {
                int processed = orderService.expireBatch(batchSize);  // own transaction per call
                total += processed;
                if (processed < batchSize) break;
            }
            log.atInfo()
               .addKeyValue("event", "order.expired-cleanup.completed")
               .addKeyValue("job_id", JOB_ID)
               .addKeyValue("processed_count", total)
               .addKeyValue("duration_ms", Duration.ofNanos(System.nanoTime() - start).toMillis())
               .log("Expired order cleanup finished");
        } catch (Exception e) {
            log.atError().setCause(e)
               .addKeyValue("event", "order.expired-cleanup.failed")
               .addKeyValue("job_id", JOB_ID)
               .addKeyValue("attempt", 1)
               .log("Expired order cleanup failed");                  // the next run retries
        }
    }
}

// OrderService: the job's only call, safe to run twice
@Transactional
public int expireBatch(int batchSize) {
    List<OrderEntity> overdue = orderRepository.findOverdueNotExpired(Instant.now(), PageRequest.of(0, batchSize));
    overdue.forEach(OrderEntity::markExpired);                        // expired rows are not selected again
    return overdue.size();
}
```

### Settings and lock table

```yaml
# application.yml
app:
  orders:
    jobs:
      expired-cleanup:
        fixed-delay: 5m
        lock-at-most-for: 10m          # longer than the longest normal run
        lock-at-least-for: 30s
        batch-size: 100
```

```java
@Validated
@ConfigurationProperties(prefix = "app.orders.jobs")
public record OrdersJobProperties(@Valid ExpiredCleanup expiredCleanup) {
    public record ExpiredCleanup(@Min(1) @Max(1000) int batchSize) {}
}
```

```sql
-- db/migration/shared/V202611161100__shared_create_shedlock.sql
CREATE TABLE shedlock (
    name       VARCHAR(64)  NOT NULL,
    lock_until TIMESTAMP    NOT NULL,
    locked_at  TIMESTAMP    NOT NULL,
    locked_by  VARCHAR(255) NOT NULL,
    CONSTRAINT pk_shedlock PRIMARY KEY (name)
);
```

### Retry around a vendor call

```java
// payments/integration/stripe/StripePaymentGateway.java
@Retryable(retryFor = TransientVendorException.class,
           maxAttempts = 3,
           backoff = @Backoff(delay = 500, multiplier = 2))
public PaymentModel charge(ChargeRequest request) { ... }
```

### Rules

1. Jobs live in `<module>/job/` and are named `<What>Job`. A job is a thin trigger: lock, one call to a service, log. It holds no business logic.
2. Schedules are never hardcoded. Values come from `application.yml` under `app.<module>.jobs.<name>`. Use `fixedDelay` by default. Use cron only for calendar times ("every day at 02:00"), in UTC.
3. `shared/config/SchedulingConfig` provides a dedicated thread pool, so jobs never block event listeners. Each run gets a new `correlationId`.
4. Every job has `@SchedulerLock` (ShedLock, table `shedlock` in PostgreSQL, created by a Flyway migration in `shared`). The lock name is the job id. `lockAtMostFor` is longer than the longest normal run.
5. Every job is safe to run twice. It selects only rows that still need work.
6. Work runs in bounded batches. The batch size comes from yml, each batch is one short transaction inside the service, and the job has a max-batches safety stop. The job loop never sits inside one transaction.
7. The next run is the retry. `@Retryable` (max 3 attempts, exponential backoff) is used only around vendor calls, and only for transient errors. Business failures are never retried.
8. A job is its own failure boundary and logs one terminal failure with the stack trace. One bad item never stops the batch.
9. Log events `<module>.<job>.started|completed|failed` with `job_id`, `attempt`, `processed_count` and `duration_ms`. Log summaries, never one line per item. An idle run logs at DEBUG.
10. Job settings are a typed `@ConfigurationProperties` record with `@Validated`, with kebab-case keys.
11. Tests: the service method (including a second run that does nothing). Lock behavior is tested later with Testcontainers.
