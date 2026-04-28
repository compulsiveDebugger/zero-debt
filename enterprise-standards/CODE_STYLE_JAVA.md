# Java 21 + Spring Boot 3 Code Style

Backend services target **Java 21 (LTS)** on **Spring Boot 3.3+**, built with **Gradle** (Kotlin DSL) or **Maven** — pick one per repo and stick with it. This document is what reviewers will hold you to.

**Tooling enforces what it can:** `spotless` (formatter), `checkstyle`, `pmd`, `error-prone`, `nullaway`, `archunit`, `spotbugs`, `jacoco`, JUnit 5. Things tooling can't enforce are below.

---

## 1. Formatting (auto-enforced)

- **`spotless` with Google Java Format (AOSP variant, 4-space indent, 100-col line length).** Never disagree with it in a PR.
- **`spotless apply`** runs as a pre-commit hook and as a required CI check.
- **One top-level type per file.** File name matches the public type.
- **Final newline, LF line endings, UTF-8.** Enforced via `.editorconfig`.

---

## 2. Language level — use Java 21 properly

Write modern Java. Reviewers will push back on pre-Java 17 idioms.

- **`record`** for immutable value objects, DTOs, query results, command/event payloads. No Lombok `@Value`/`@Data` for new code.
- **Sealed types + pattern matching** for closed hierarchies (results, domain events, state machines).
- **`switch` expressions** with pattern matching — exhaustive, no fall-through.
- **`var`** for local variables when the right-hand side makes the type obvious. Never on public API; never when the type adds reading value.
- **Text blocks (`"""`)** for SQL, JSON fixtures, multi-line strings. Strip indentation with `.stripIndent()` only when needed.
- **Virtual threads** are the default for blocking I/O — see §10.
- **`Optional`** for return types only. Never as a parameter, never as a field.
- **Streams** when they read better than a loop; otherwise use a loop. Don't force a stream that needs `.collect(toList()).get(0)`.

```java
public sealed interface PaymentResult
    permits PaymentResult.Approved, PaymentResult.Declined, PaymentResult.Pending {

  record Approved(UUID transactionId, Money captured)        implements PaymentResult {}
  record Declined(DeclineReason reason, String issuerCode)   implements PaymentResult {}
  record Pending(UUID transactionId, Instant retryAfter)     implements PaymentResult {}
}

String describe(PaymentResult r) {
  return switch (r) {
    case PaymentResult.Approved a -> "approved " + a.captured();
    case PaymentResult.Declined d -> "declined: " + d.reason();
    case PaymentResult.Pending  p -> "pending until " + p.retryAfter();
  };
}
```

---

## 3. Naming

| Kind | Convention | Example |
|---|---|---|
| Package | lowercase, no underscores | `com.acme.billing.invoice` |
| Class / Record / Interface | `PascalCase` | `InvoiceService`, `LineItem` |
| Method / Field / Local | `camelCase` | `calculateTax` |
| Constant (`static final`) | `UPPER_SNAKE_CASE` | `MAX_RETRY_ATTEMPTS` |
| Generic type param | one capital letter, or `PascalCase` if needed | `T`, `EventT` |
| Test class | `<Subject>Test` | `InvoiceServiceTest` |
| Test method | `methodUnderTest_condition_expected` | `calculateTax_zeroQuantity_returnsZero` |

Avoid:
- Hungarian prefixes (`strName`, `iCount`).
- `I` prefix on interfaces. The interface is the noun (`PaymentGateway`), the impl gets the modifier (`StripePaymentGateway`).
- `Impl` suffix when there's only one implementation — pick a real name. `Impl` is allowed when the interface has multiple legitimate implementations.
- Vague nouns: `Manager`, `Helper`, `Util`, `Processor`, `Handler` without a qualifier. If you can't name it more specifically, the abstraction is wrong.
- Single-letter names except trivially scoped iterators (`i`, `n`) and generic type parameters.

---

## 4. Package structure

Organize **by feature**, not by layer.

```
com.acme.billing
├── invoice/
│   ├── InvoiceController.java
│   ├── InvoiceService.java
│   ├── InvoiceRepository.java
│   ├── domain/        ← entities, records, value objects
│   ├── api/           ← request/response DTOs, OpenAPI annotations
│   └── internal/      ← package-private helpers
├── payment/
└── shared/            ← cross-feature primitives only (Money, TenantId, …)
```

- **No `controller/`, `service/`, `repository/` top-level packages.** Layer-by-feature scales; layer-by-type doesn't.
- **`internal` sub-package = package-private boundary.** Enforced via ArchUnit.
- **No cyclic dependencies between features.** ArchUnit test required.

---

## 5. Classes & methods

- **Default to `final`** on classes and fields. Only remove `final` when subclassing or mutation is required.
- **Constructor injection only.** Never `@Autowired` on a field. Records and `@RequiredArgsConstructor` (Lombok) both fine.
- **Single responsibility.** If the class name needs "and", split it.
- **Methods ≤40 lines, ≤4 parameters.** Beyond that, extract a method or introduce a parameter object (a `record`).
- **Public methods read top-down**: public above private, callers above callees.
- **No static mutable state.** Static utility classes are `final` with a private constructor and only pure functions.

---

## 6. Nullability

- **`@NullMarked` at the package level** (JSpecify). Everything is non-null unless annotated `@Nullable`.
- **NullAway runs in CI** and fails the build on violations.
- **Never return `null` from a collection-returning method.** Return an empty collection.
- **`Optional<T>`** for return types where absence is a normal outcome. Don't use `Optional` as a field or parameter type — it's a code smell that signals an unclear API.
- **`Objects.requireNonNull(x, "x")`** at the top of public methods that must reject null.

---

## 7. Records, DTOs, and domain objects

- **Records for DTOs and value objects.** Compact constructor for validation.
- **Don't expose JPA entities across the API boundary.** Map to a `record` DTO.
- **Use [MapStruct](https://mapstruct.org/)** for entity ↔ DTO mapping. No hand-written 30-line mappers.
- **Domain objects encapsulate invariants.** A `Money` should reject negative amounts in its constructor, not in every caller.

```java
public record Money(BigDecimal amount, Currency currency) {
  public Money {
    Objects.requireNonNull(amount, "amount");
    Objects.requireNonNull(currency, "currency");
    if (amount.scale() > currency.getDefaultFractionDigits()) {
      throw new IllegalArgumentException("scale exceeds currency precision");
    }
  }
}
```

---

## 8. Spring Boot conventions

### 8.1 Configuration

- **`@ConfigurationProperties` records** for all external config. No `@Value` injection in business code.
- **`application.yml`** is the source of truth; profile overrides in `application-<profile>.yml`. No `application.properties`.
- **Validate on startup:** `@Validated` on `@ConfigurationProperties` plus `@NotNull`, `@Min`, etc. The app should refuse to start with bad config.
- **Secrets never in YAML.** Use Vault / AWS Secrets Manager / Azure Key Vault via Spring Cloud Config. Local dev uses `.env` ignored by git.
- **`application-test.yml`** for tests; never overload the `default` profile.

### 8.2 Web layer (Spring MVC)

- **`@RestController` per resource.** Thin: validate input, call service, map to response.
- **No business logic in controllers.** A controller method is ≤15 lines.
- **`@Valid` on every request body.** Bean Validation (Jakarta) annotations on the DTO record components.
- **Return `ResponseEntity<T>`** when you need to set status/headers; return `T` directly when 200/201 is implicit.
- **Pagination via `Pageable`** — `Page<T>` responses follow the shape in `API_STYLE.md`.
- **OpenAPI annotations live on the DTO** (`@Schema`) and controller (`@Operation`). The spec is generated, not hand-written.

### 8.3 Service layer

- **`@Service` is annotation-only.** Don't use it as a marker for "things with business logic" if it isn't actually injected.
- **Stateless.** Any state belongs in a repository, cache, or `ApplicationContext` bean with explicit scope.
- **Transaction boundaries on services**, not repositories: `@Transactional` at the public service method.
- **`@Transactional(readOnly = true)`** for reads — the default should be read-only at class level, with method-level overrides for writes.

### 8.4 Persistence

- **Spring Data JPA** for CRUD; **`JdbcClient` / jOOQ** for non-trivial queries. No native SQL strings scattered in repositories.
- **Flyway** for schema migrations. Never `ddl-auto: update` outside `local`.
- **No `EAGER` fetching** on associations. Default is `LAZY`; use entity graphs / fetch joins for the read paths that need them.
- **No JPA entity in the controller signature.** The boundary is a DTO.
- **Auditing** via `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy`.

### 8.5 Bean wiring

- **Constructor injection always.** No field, no setter injection.
- **Avoid `@ComponentScan` widening.** The default scan from the `@SpringBootApplication` package is enough.
- **`@Primary` is a smell.** Prefer `@Qualifier` with explicit names.
- **`@ConditionalOnProperty`** for feature flags at the bean-graph level. Runtime flags use the feature-flag service.

---

## 9. Errors

- **Define a small set of explicit exceptions per domain** — `InvoiceNotFoundException`, `PaymentDeclinedException`, `TenantQuotaExceededException`. Never throw raw `RuntimeException` from production code.
- **Exceptions extend a base `DomainException`** with an error-code enum and structured payload.
- **One `@RestControllerAdvice`** maps domain exceptions to the standard error envelope (see `API_STYLE.md`). Controllers don't catch.
- **Catch the narrowest exception** that makes sense. `catch (Exception e)` only at framework boundaries.
- **Re-throw with cause:** `throw new InvoiceLockedException(id, e)` — never lose the original.
- **Never swallow.** Every `catch` either rethrows, logs at WARN+ with context, or has a comment explaining why swallowing is correct.
- **No checked exceptions in new code.** Wrap them at the boundary.

---

## 10. Concurrency

- **Virtual threads (`Executors.newVirtualThreadPerTaskExecutor()`)** are the default for blocking I/O. Set `spring.threads.virtual.enabled=true`.
- **Don't pin virtual threads.** No `synchronized` around blocking I/O — use `ReentrantLock`. ErrorProne/PMD checks watch for this.
- **Never block in a reactive (`Mono`/`Flux`) chain.** If the codebase mixes blocking and reactive, isolate them.
- **Bound everything.** No unbounded `CompletableFuture.allOf` over user input. Use a `Semaphore` or a sized executor.
- **No `Thread.sleep`** in production code outside well-justified retry/backoff helpers — and even those go through Resilience4j.
- **No mutable static state.** Per-request data goes in Spring's request scope or `ThreadLocal` cleared in a filter.

---

## 11. Logging

Full conventions in `LOGGING_STYLE.md`. Quick rules:

- **SLF4J only** (`org.slf4j.Logger`). Never `System.out`, never log4j directly.
- **Parameterized messages:** `log.info("invoice paid id={} amount={}", id, amount)`. Never string-concatenate.
- **Structured fields via MDC:** `tenantId`, `requestId`, `userId` — set in a filter, propagated automatically.
- **Lowercase event-style messages** for searchability: `"invoice_paid"`, not `"Invoice has been paid successfully!"`.
- **No PII / secrets.** A redaction layer catches the obvious; you're still responsible.
- **Log levels:**
  - `ERROR` — actionable, oncall is paged.
  - `WARN` — degraded behavior, retry succeeded, validation rejection.
  - `INFO` — lifecycle events (startup, key business transitions).
  - `DEBUG` — developer detail; off in prod.
  - `TRACE` — wire-level; never enabled in prod.

---

## 12. Testing

Full guide in `TESTING.md`. Style highlights:

- **JUnit 5 + AssertJ + Mockito.** No JUnit 4, no Hamcrest, no PowerMock.
- **`@SpringBootTest` is the last resort.** Prefer pure unit tests, then `@WebMvcTest` / `@DataJpaTest` slices.
- **Testcontainers** for integration tests against Postgres / Kafka / Redis. Never H2 to "fake" Postgres.
- **One behavior per test.** Multiple assertions are fine if they describe the same behavior — group them with `assertSoftly`.
- **Arrange-Act-Assert** structure with blank lines between sections.
- **No `@Disabled` without a linked ticket.**
- **Coverage floor: 80% line, 70% branch** on `service/` and `domain/`. Controllers and config are excluded.
- **Mutation testing (`pitest`)** runs nightly; PRs that drop the score >2 points need justification.

---

## 13. Build & dependencies

- **Single JDK version pinned in `.tool-versions` / `.sdkmanrc`.** CI uses the same.
- **`gradle/libs.versions.toml`** (or Maven BOM) — every version in one place. No inline `"5.10.0"` in build scripts.
- **Renovate / Dependabot** weekly. Major bumps get an ADR.
- **Forbidden dependencies** enforced via `gradle-enforcer` / `maven-enforcer`: log4j 1.x, commons-logging, joda-time, javax.* (post-Jakarta).
- **No `SNAPSHOT` versions** in `main`. Release branches only.
- **Reproducible builds:** `--no-daemon` in CI, locked dependencies via `dependencies.lock`.

---

## 14. API design

Full conventions in `API_STYLE.md`. The hard rules:

- **Versioning at the URL path** (`/api/v1/invoices`). Bump only on a breaking change.
- **Plural resource nouns**, lowercase, kebab-case.
- **Standard error envelope** — RFC 7807 `application/problem+json`.
- **No verbs in URLs** except for explicitly modeled actions (`POST /invoices/{id}/void`).
- **Idempotency keys** on `POST` endpoints that create externally-visible side effects.
- **Pagination:** cursor-based for unbounded collections; offset-based only for admin tools.

---

## 15. Security

Full checklist in `SECURITY.md`. The non-negotiables:

- **Spring Security on every endpoint.** Default-deny; explicit `permitAll()` for public ones.
- **OAuth2 Resource Server** for service-to-service. JWT validated against the IdP's JWKS.
- **No raw SQL string concatenation.** Parameterized queries always.
- **CSRF on cookie-based browser flows;** disabled only when JWT is the sole credential.
- **Output encoding** on every templated response. Use Thymeleaf's auto-escaping; never disable.
- **Secrets** loaded at startup; never logged, never in stack traces (filter via `MaskingPatternLayout`).
- **Dependency scanning** (`gradle dependencyCheck` / OWASP) runs on every PR.

---

## 16. Observability

- **Micrometer** for metrics. Every public service method gets a timer (`@Timed` or programmatic).
- **OpenTelemetry** for traces. Auto-instrumentation covers HTTP/DB; manual spans for business operations.
- **Health endpoints** (`/actuator/health/{liveness,readiness}`) implemented per service. Don't expose `/actuator/*` publicly.
- **Standard metric names** — `<domain>.<entity>.<action>.{count,duration}`, e.g. `billing.invoice.create.duration`.

---

## 17. Comments and Javadoc

- **Default to no comments.** Names should carry the meaning.
- **Comments explain WHY**, never WHAT. A workaround, a perf hack, a rule from an external system, an invariant the type system can't capture.
- **Javadoc on public API of library / shared modules** — one paragraph contract. Skip `@param` / `@return` if types and names are self-evident.
- **No Javadoc on getters, setters, or trivially named methods.**
- **No `// TODO` without a linked ticket** — `// TODO(JIRA-1234): switch to v2 client once X`.
- **Never reference the current PR or task** in code or comments — those rot.

---

## 18. Don'ts (review will reject)

- `System.out.println` / `System.err.println` in production code.
- `e.printStackTrace()` — log it properly or rethrow.
- Catching `Throwable` or `Error`.
- `@SuppressWarnings` without a comment justifying the suppression.
- `@Autowired` field injection.
- `new RestTemplate()` — use the configured `WebClient` / `RestClient` bean.
- `Date`, `Calendar`, `SimpleDateFormat` — `java.time` only.
- `Vector`, `Hashtable`, `Stack` — use `ArrayList`, `HashMap`, `Deque`.
- Mutable public fields.
- `static { ... }` initializers that do I/O or reflection.
- Lombok `@Data` / `@AllArgsConstructor` on entities — they break JPA equality and unsafe-publish mutable state. `@Getter` + explicit constructor is fine.
- Mixing `javax.*` and `jakarta.*` imports.
- Reflection in business code.
- Returning `null` from a method whose name reads like it should return data — return `Optional` or empty collection.

---

## 19. Recommended reading

- *Effective Java*, 3rd ed. (Bloch) — still the baseline.
- *Java Concurrency in Practice* (Goetz) — read chapters 1–6 before writing concurrent code.
- [Spring Boot Reference](https://docs.spring.io/spring-boot/reference/) — the relevant version, not whatever Google surfaces.
- [JEP index](https://openjdk.org/jeps/0) — read the JEPs for the language features you use (records, sealed types, pattern matching, virtual threads).
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html) — for the rare cases Spotless doesn't cover.
- [JSpecify](https://jspecify.dev/) — nullability annotations.
