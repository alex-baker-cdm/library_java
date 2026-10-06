# API design & security standards audit - alex-baker-cdm/library_java (2026-10-06)

## Summary

| | |
|---|---|
| Repository / classification | `alex-baker-cdm/library_java` - **provider** (Spring Boot 2.2.0.M6, Java 11, DDD sample "library"). `alex-baker-cdm/library_demo` is reported by the orchestrator as a byte-identical copy: **every finding below applies to both repositories.** |
| Surface | REST (HAL / HAL-FORMS over JSON), 7 application routes + Spring Boot Actuator |
| Specification | **None** - no OpenAPI / GraphQL / proto file in the repo |
| Auth model | **None** - no Spring Security, no API key, no principal; the `patronId` path segment is the only identity |
| Runtime verified | **Yes** - service run locally with the `local` profile (in-memory H2, seeded book/branch/patron) and 38 curl probes recorded (T01-T38) |
| Result | 13 findings: 1 critical, 4 high, 6 medium, 2 low; 7 positives; 7 items needing confirmation |

**How it was run.** `JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64 mvn compile` fails on `master` for two pre-existing reasons: duplicate classes in `src/main/java/io/pillopl/library/lending/book/new_model/` and the unrelated IGV GUI sources in `src/main/java/io/pillopl/library/ui/`. Those two directories were moved out of the tree **temporarily and locally** (never committed; restored before this report was committed), after which `mvn compile`, `mvn test` (89 Spock tests run, 0 failures, 0 errors, 0 skipped) and `mvn spring-boot:run` with `SPRING_PROFILES_ACTIVE=local` worked. repo1.maven.org answered HTTP 429 to direct downloads from this VM, so a temporary local mirror settings file was used instead of the orchestrator's `/tmp/empty-settings.xml`. The Docker/compose monitoring stack was **not** started; area F statements about it are based on the committed files. No application code, config or dependency was changed.

**What was verified at runtime.** Unauthenticated read + write + delete on patron resources and IDOR by swapping `patronId` (T01-T11, T36-T38); duplicate / unknown-patron / malformed / empty / wrong-type / negative / Integer.MAX_VALUE bodies (T07, T13-T19); non-UUID path, unknown route, wrong method, wrong content type (T20-T23); CORS pre-flight and response headers (T24-T25); every actuator endpoint (T26-T32); a 12 MB body (T33); checkouts collection (T34-T35). The full transcript was kept in the session (`/tmp/transcript.txt`) and the relevant lines are quoted under each finding.

## Mounted route inventory

All handlers live in `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java` (`@RestController`, line 47). No outbound HTTP calls exist (provider only).

| Method | Route | Handler (line) | Auth | Notes |
|---|---|---|---|---|
| GET | `/profiles/{patronId}` | `patronProfile` (55) | none | returns HAL links only; any UUID -> 200 |
| GET | `/profiles/{patronId}/holds/` | `findHolds` (60) | none | trailing slash required (no slash -> 405) |
| GET | `/profiles/{patronId}/holds/{bookId}` | `findHold` (72) | none | 404 when absent |
| GET | `/profiles/{patronId}/checkouts/` | `findCheckouts` (81) | none | self link points at `/holds/` (bug) |
| GET | `/profiles/{patronId}/checkouts/{bookId}` | `findCheckout` (92) | none | 404 when absent |
| POST | `/profiles/{patronId}/holds` | `placeHold` (100) | none | 200 empty body on success; 500 on domain rejection |
| DELETE | `/profiles/{patronId}/holds/{bookId}` | `cancelHold` (116) | none | 204 / 404 |
| GET | `/actuator`, `/actuator/health`, `/actuator/info`, `/actuator/metrics`, `/actuator/metrics/{name}`, `/actuator/prometheus` | Spring Boot Actuator (`application.yml:1-5`) | none | env/beans/mappings/heapdump -> 404 |

Checkout and return are domain operations (`CheckingOut`, `Returning`) with **no REST endpoint**; they are driven only by tests.

## Findings by area (A-G)

| ID | Severity | Title | File:lines | Effort |
|---|---|---|---|---|
| LIB-A-01 | medium | No API specification; mounted contract has inconsistent trailing-slash routes and a wrong HATEOAS self link | `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:55-118` | M |
| LIB-A-02 | medium | POST /holds is non-idempotent and maps domain outcomes to 200/500 instead of 201/409/404 | `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:100-114` | M |
| LIB-B-01 | critical | No authentication or object-level authorization: any caller can read and mutate any patron's holds | `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:47-118` | M |
| LIB-C-01 | high | No edge validation on PlaceHoldRequest; nulls/negatives become 500 and Integer.MAX_VALUE creates a hold until year 5881637 | `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:100-114,184-188` | S |
| LIB-C-02 | medium | No request body size limit: a 12 MB JSON body is fully read and parsed | `src/main/resources/application.yml:1-5` | S |
| LIB-D-01 | high | Default Spring error pages leak internals (Jackson, Lombok, converter messages) and there is no single error envelope | `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:100-118` | M |
| LIB-D-02 | high | Cancelling a hold returns 204 but the holds read model is never updated (missing @EventListener on BookHoldCanceled) | `src/main/java/io/pillopl/library/lending/dailysheet/infrastructure/SheetsReadModel.java:113-119` | S |
| LIB-E-01 | medium | Actuator metrics and Prometheus scrape endpoint are reachable without authentication on the application port | `src/main/resources/application.yml:1-5` | S |
| LIB-E-02 | low | Logs are unstructured Logback defaults with no request/correlation id, and no request id is returned to callers | `src/main/java/io/pillopl/library/lending/patron/application/hold/PlacingOnHold.java:28-37` | M |
| LIB-F-01 | high | Runtime dependency tree carries 27 critical and 92 high advisories with fixes available (Boot 2.2.0.M6 milestone, Jackson 2.9.9.3, Tomcat 9.0.24, H2 1.4.197) | `pom.xml:6-8,19,86-88,125-127` | L |
| LIB-F-02 | medium | Grafana admin password committed in monitoring config and monitoring stack ships on end-of-life images | `monitoring/grafana/config.monitoring:1-2` | S |
| LIB-F-03 | low | No CORS policy, security headers or rate limiting on the API | `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:47-54` | S |
| LIB-G-01 | medium | CI gate is ineffective: master does not compile, and there are no contract tests, spec lint, changelog or deprecation markers | `.circleci/config.yml:23-33` | M |
Area coverage: A contract/design (A-01, A-02) - B authn/authz (B-01) - C input validation (C-01, C-02) - D error handling (D-01, D-02) - E logging/observability (E-01, E-02) - F security hygiene (F-01, F-02, F-03) - G lifecycle (G-01). No money-moving endpoints exist, so numeric money handling was not applicable.

## Detailed findings

### LIB-A-01 - medium - No API specification; mounted contract has inconsistent trailing-slash routes and a wrong HATEOAS self link
**File:** `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:55-118` - **Effort:** M

**Evidence.** Repo search found no OpenAPI/GraphQL/proto file (only pom.xml, README, Spock specs). Runtime: `curl GET /profiles/{id}/holds/` -> 200 but `curl GET /profiles/{id}/holds` -> 405 "Request method 'GET' not supported" while `POST /profiles/{id}/holds` (no slash) is the create route (T03/T04). `curl GET /profiles/{id}/checkouts/` -> 200 with body `{"_links":{"self":{"href":".../holds/"}}}` - the checkouts collection advertises the holds URL as its self link (T34; line 89 uses `findHolds` in `linkTo`). No versioning (no /v1, no media-type version); git history of the controller shows the placeHold route and response shape changed (commits 84e5774, c88c538) with no version bump.

**Remediation.** Publish an OpenAPI 3.1 document (springdoc-openapi) generated from the controllers, lint it with Spectral in CI, and add a contract test that diffs mounted `RequestMappingHandlerMapping` routes against the spec. Normalise collection routes to a single form without trailing slash (`/profiles/{id}/holds` for GET and POST) and fix the checkouts self link to `findCheckouts`. Adopt URI versioning (`/v1/...`) or a documented deprecation policy before further route changes.

### LIB-A-02 - medium - POST /holds is non-idempotent and maps domain outcomes to 200/500 instead of 201/409/404
**File:** `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:100-114` - **Effort:** M

**Evidence.** `curl -X POST .../profiles/{patron}/holds -d '{bookId,libraryBranchId,numberOfDays:3}'` -> 200 with empty body and no Location header (T04). Re-sending the identical request -> 500 (T07); app log: `IllegalArgumentException: Cannot find available book with Id: 997cbcce-...` thrown by PlacingOnHold.java:53 and surfaced as 500. POST for an unknown patron id -> 500 (T13; log `Patron with given Id does not exists`). Controller lines 103-114 only handle the Either success/BookHoldFailed branches; `Try` failures fall through to the default 500.

**Remediation.** Return 201 Created with a Location header for a new hold; map `Cannot find available book` to 409 Conflict and unknown patron/book to 404 via a `@RestControllerAdvice` that translates `Try` failures and `BookHoldFailed` to RFC 9457 problem+json. Support an `Idempotency-Key` header (stored in a keyed table with the response) so retries of a hold return the original result instead of 500.

### LIB-B-01 - critical - No authentication or object-level authorization: any caller can read and mutate any patron's holds
**File:** `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:47-118` - **Effort:** M

**Evidence.** No Spring Security dependency in pom.xml, no filter/interceptor, no principal is read anywhere in the controller. Runtime, with no credentials: `curl GET /profiles/56b560fe-...` -> 200 with profile links (T01); `curl GET /profiles/11111111-1111-1111-1111-111111111111` -> 200 (any UUID yields a profile, T08); `curl -X POST /profiles/56b560fe-.../holds -d {...}` -> 200 and a hold was created (T04); `curl -X DELETE /profiles/56b560fe-.../holds/997cbcce-...` -> 204 and the victim's hold was cancelled (T10, confirmed by a later POST for the same book succeeding, T18). The patronId path segment is the only 'identity'.

**Remediation.** Add spring-boot-starter-oauth2-resource-server (JWT validated against the org IdP, or mTLS at the gateway with a propagated identity header) and a `SecurityFilterChain` that requires authentication on `/profiles/**`. Enforce object-level authorization by resolving the patron from the authenticated principal (or checking `principal.patronId == pathPatronId` / librarian role) in a `@PreAuthorize` expression or a domain-level `PatronAccessPolicy`, and cover it with a Spock test that asserts 403 for a foreign patronId.

### LIB-C-01 - high - No edge validation on PlaceHoldRequest; nulls/negatives become 500 and Integer.MAX_VALUE creates a hold until year 5881637
**File:** `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:100-114,184-188` - **Effort:** S

**Evidence.** `PlaceHoldRequest` (lines 184-188) is `UUID bookId; UUID libraryBranchId; Integer numberOfDays;` with no Bean Validation annotations and `@RequestBody` has no `@Valid`; spring-boot-starter-validation is not a dependency. Runtime: `curl -X POST ... -d '{}'` -> 500 `"message":"libraryBranchId is marked @NonNull but is null"` (T15); `numberOfDays:-5` -> 500 (T17; log `Cannot use negative integer or zero as number of days`); `numberOfDays:2147483647` -> 200 (T18) and `GET /holds/` then lists a hold with `"till":"+5881637-04-15T08:18:17.105606Z"` (T19).

**Remediation.** Add spring-boot-starter-validation, annotate `PlaceHoldRequest` with `@NotNull` on both UUIDs and `@NotNull @Min(1) @Max(<policy max, e.g. 30>)` on numberOfDays, use `@Valid @RequestBody`, and map `MethodArgumentNotValidException` to a 400 problem+json with per-field errors. Keep the domain `HoldDuration` guard as defence in depth but bound it with an upper limit.

### LIB-C-02 - medium - No request body size limit: a 12 MB JSON body is fully read and parsed
**File:** `src/main/resources/application.yml:1-5` - **Effort:** S

**Evidence.** application.yml contains only actuator exposure; no `server.tomcat.max-swallow-size`, `spring.servlet.multipart` or filter-based limit exists (grep over src/main/resources and pom.xml). Runtime: `curl -X POST -H 'Content-Type: application/json' --data-binary @/tmp/big.json` (12,583,045 bytes, valid JSON with the hold fields plus a 12 MB padding field) -> 500 after full parse; the log shows the request reached the domain (`Cannot find available book ...`), i.e. nothing rejected the body at the edge (T33).

**Remediation.** Reject oversized bodies at the edge: a `OncePerRequestFilter`/Tomcat connector `maxPostSize` + `maxSwallowSize` tuned to a few KB for these endpoints (or the API gateway's body-size limit), returning 413 Payload Too Large as a problem+json response. Configure Jackson `StreamReadConstraints` (available after upgrading Jackson, see LIB-F-01) and `FAIL_ON_UNKNOWN_PROPERTIES` so unknown padding fields are rejected.

### LIB-D-01 - high - Default Spring error pages leak internals (Jackson, Lombok, converter messages) and there is no single error envelope
**File:** `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:100-118` - **Effort:** M

**Evidence.** No `@ControllerAdvice`/`ErrorController`/problem library exists in the repo. Runtime bodies: malformed JSON -> 400 `"message":"JSON parse error: Unexpected character ('b' (code 98))... nested exception is com.fasterxml.jackson.core.JsonParseException ... at [Source: (PushbackInputStream); line: 1, column: 3]"` (T14); `{}` -> 500 `"message":"libraryBranchId is marked @NonNull but is null"` (T15, Lombok-generated message naming the internal field); `/profiles/not-a-uuid` -> 400 `"Failed to convert value of type 'java.lang.String' to required type 'java.util.UUID'; nested exception is java.lang.IllegalArgumentException: Invalid UUID string"` (T20). Domain failures (T07, T13, T17) return 500 with an empty body, successes return 200 with an empty body, and 404s (T12, T35) are empty while unknown routes return the Spring `{timestamp,status,error,message,path}` shape (T21) - three different shapes.

**Remediation.** Introduce one `@RestControllerAdvice` producing RFC 9457 `application/problem+json` (`type`, `title`, `status`, `detail`, `instance`, plus `code` and `correlationId`) for `HttpMessageNotReadableException`, `MethodArgumentTypeMismatchException`, validation errors and domain `Try` failures; set `server.error.include-message=never` / `include-stacktrace=never` and disable the whitelabel fallback so no framework message reaches clients.

### LIB-D-02 - high - Cancelling a hold returns 204 but the holds read model is never updated (missing @EventListener on BookHoldCanceled)
**File:** `src/main/java/io/pillopl/library/lending/dailysheet/infrastructure/SheetsReadModel.java:113-119` - **Effort:** S

**Evidence.** `handle(BookPlacedOnHold)`, `handle(BookHoldExpired)`, `handle(BookCheckedOut)` are annotated `@EventListener` (lines 90, 122, 131) but `handle(BookHoldCanceled)` at line 113-119 is not, so the `UPDATE holds_sheet SET canceled_at=..., status='CANCELED'` never runs. Runtime: `curl -X DELETE /profiles/56b.../holds/997c...` -> 204 (T10) yet `GET /profiles/56b.../holds/` still lists the hold with a `delete` affordance (T11). After a second hold on the same book, two DELETEs returned 204/204 and the third 404 (T36-T38, aggregate state cancelled) while `GET /holds/` still listed both holds (T37). The write side (Patron aggregate + Book) and the read side (`holds_sheet`, which backs `PatronProfiles`) are permanently inconsistent after any cancel.

**Remediation.** Annotate `handle(BookHoldCanceled)` with `@EventListener` (and `@Transactional` like the others) and add a Spock scenario `cancelled hold disappears from /holds`. Longer term, standardise projections on a transactional outbox / `@TransactionalEventListener(AFTER_COMMIT)` with a replayable event log so a dropped handler is detected by a consistency check rather than by users.

### LIB-E-01 - medium - Actuator metrics and Prometheus scrape endpoint are reachable without authentication on the application port
**File:** `src/main/resources/application.yml:1-5` - **Effort:** S

**Evidence.** `management.endpoints.web.exposure.include: info, health, metrics, prometheus` with no `management.server.port` or security. Runtime: `curl GET /actuator/metrics` -> 200 `{"names":["jvm.threads.states","http.server.requests","domain_events",...]}` (T28); `curl GET /actuator/prometheus` -> 200 with full JVM, Tomcat and `http_server_requests_seconds` series including URI/status labels (T29); `curl GET /actuator` -> 200 discovery index (T26).

**Remediation.** Move management endpoints to a separate `management.server.port` bound to the cluster-internal network (or protect them with the same `SecurityFilterChain` and a `METRICS` role used by the Prometheus scraper), leave only `/actuator/health/{liveness,readiness}` public, and disable the `/actuator` discovery page.

### LIB-E-02 - low - Logs are unstructured Logback defaults with no request/correlation id, and no request id is returned to callers
**File:** `src/main/java/io/pillopl/library/lending/patron/application/hold/PlacingOnHold.java:28-37` - **Effort:** M

**Evidence.** No logback-spring.xml or JSON encoder in src/main/resources; log lines are the Spring default text pattern (`2026-10-06 08:18:16.994 ERROR 19601 --- [nio-8080-exec-3] i.p.l.l.p.a.hold.PlacingOnHold : Failed to place a hold`). Runtime response headers for a 200 are only `Content-Type`, `Transfer-Encoding`, `Date` - no `X-Request-Id`/`traceparent` (T25). Domain failures are logged at ERROR with full stack traces for expected business outcomes (duplicate hold, negative days), while the NPE from T15 is logged only by Tomcat's `Servlet.service()` handler, so 5xx cannot be distinguished from business noise by log level alone. No access log is configured.

**Remediation.** Add logstash-logback-encoder (JSON) with MDC fields `traceId`/`spanId` from Micrometer Tracing / Sleuth (Boot 2.x) or OpenTelemetry, return the id in an `X-Request-Id` header via a filter, log expected domain rejections at WARN and only unexpected exceptions at ERROR, and emit the status code and route in a structured access log so 5xx rates are alertable.

### LIB-F-01 - high - Runtime dependency tree carries 27 critical and 92 high advisories with fixes available (Boot 2.2.0.M6 milestone, Jackson 2.9.9.3, Tomcat 9.0.24, H2 1.4.197)
**File:** `pom.xml:6-8,19,86-88,125-127` - **Effort:** L

**Evidence.** `mvn dependency:list` (runtime scope, 67 artifacts) fed to the OSV API (osv-scanner 2.6.0 could not reach Maven Central: HTTP 429) returned 183 advisories; filtering to GitHub-severity HIGH/CRITICAL with a fixed version: 22 packages, 27 critical + 92 high. Worst: com.fasterxml.jackson.core:jackson-databind 2.9.9.3 (11 critical/39 high, e.g. GHSA-4w82-r329-3q67, fixed >=2.9.10.x), org.apache.tomcat.embed:tomcat-embed-core 9.0.24 (8 critical/23 high, e.g. GHSA-5m62-pw8w-7w9f), org.springframework:spring-webmvc/spring-web/spring-beans 5.2.0.RC2 (Spring4Shell GHSA-36p3-wjmg-h94x, fixed 5.3.18), com.h2database:h2 1.4.197 (2 critical, GHSA-45hx-wfhj-473x, fixed 2.1.210), org.yaml:snakeyaml 1.25 (3 high, fixed 2.0), spring-hateoas 1.0.0.BUILD-SNAPSHOT from repo.spring.io/snapshot (unreproducible build). Parent is a pre-release milestone `2.2.0.M6` (pom.xml:8).

**Remediation.** Move to a supported Spring Boot GA line (2.7.x minimum, ideally 3.3.x) using the Boot BOM for all Spring/Jackson/Tomcat/SnakeYAML versions, pin spring-hateoas to a GA release and drop the snapshot repositories, upgrade H2 to 2.x (test scope only), and add `osv-scanner`/`dependency-check` to the CircleCI pipeline failing on high/critical advisories that have a fix.

### LIB-F-02 - medium - Grafana admin password committed in monitoring config and monitoring stack ships on end-of-life images
**File:** `monitoring/grafana/config.monitoring:1-2` - **Effort:** S

**Evidence.** `monitoring/grafana/config.monitoring:1` is `GF_SECURITY_ADMIN_PASSWORD=password`, loaded by `docker-compose.yml` (`env_file`, grafana service lines 23-27) which publishes Grafana on 3000 and Prometheus (`prom/prometheus:v2.4.3`, line 15) on 9090 to the host. Runtime images: `Dockerfile:1` and `Dockerfile.build:6` use `openjdk:11.0.1-jre-slim-stretch` (Debian stretch EOL, JDK 11.0.1 from 2018). Not executed at runtime (Docker stack not started on this VM); evidence is the committed configuration.

**Remediation.** Remove the password from git, inject `GF_SECURITY_ADMIN_PASSWORD` from a secret store (compose `secrets:`/env from vault) and rotate it; pin the app image to a maintained base (`eclipse-temurin:17-jre` or distroless) and current Prometheus/Grafana tags; add gitleaks or trufflehog to CI to block future committed credentials.

### LIB-F-03 - low - No CORS policy, security headers or rate limiting on the API
**File:** `src/main/java/io/pillopl/library/lending/patronprofile/web/PatronProfileController.java:47-54` - **Effort:** S

**Evidence.** No `CorsConfiguration`/`@CrossOrigin`, no Spring Security headers (`X-Content-Type-Options`, `Cache-Control`, `Strict-Transport-Security`), no Bucket4j/Resilience4j rate limiter in pom.xml. Runtime: `curl -X OPTIONS -H 'Origin: https://evil.example' -H 'Access-Control-Request-Method: POST' .../holds` -> 400 with empty body (T24, pre-flight unhandled rather than explicitly denied); `curl -I GET /profiles/{id}` -> only `Content-Type: application/prs.hal-forms+json`, `Transfer-Encoding`, `Date` (T25).

**Remediation.** Once Spring Security is added (LIB-B-01) its defaults provide `X-Content-Type-Options`, `Cache-Control`, `X-Frame-Options`/`HSTS`; define an explicit `CorsConfigurationSource` allow-list for the known front-end origins, and add a per-principal rate limit (Bucket4j filter or gateway policy) returning 429 with `Retry-After`.

### LIB-G-01 - medium - CI gate is ineffective: master does not compile, and there are no contract tests, spec lint, changelog or deprecation markers
**File:** `.circleci/config.yml:23-33` - **Effort:** M

**Evidence.** `.circleci/config.yml:23` runs `./mvnw verify` then codecov, but `mvn compile` on master fails with duplicate class errors (`src/main/java/io/pillopl/library/lending/book/new_model/*` duplicates `lending/book/model/*`) and ~hundreds of unresolved-symbol errors in `src/main/java/io/pillopl/library/ui/` (IGV GUI sources unrelated to the service); the service could only be run after temporarily moving those two directories out of the tree. No CHANGELOG, no `@Deprecated`/`Sunset`/`Deprecation` headers, no Spectral/OpenAPI lint, no contract or consumer-driven tests; deployment is a hand-run `docker-compose` with `spring/library:latest` and no rollback procedure documented in README.

**Remediation.** Make the build green (remove the stray `ui/` and `new_model` sources or move them to a separate module) and make CircleCI a required status check on master; add a contract-test stage (spec diff + Spectral ruleset) and a `CHANGELOG.md` with a documented deprecation policy (`Deprecation`/`Sunset` headers per RFC 8594); publish immutable, version-tagged images instead of `latest` so rollback is a tag change.
## Positives (candidate reference implementations)

- All SQL in the read models and repositories is parameterised JdbcTemplate (`SheetsReadModel`, `PatronProfileReadModel`, `BookDatabaseRepository`): no string concatenation into queries was found and UUID path variables are typed, so no SQL injection surface was reachable (T16/T20 are rejected before any query).
- Actuator exposure is an explicit allow-list (`info, health, metrics, prometheus`); `/actuator/env`, `/actuator/heapdump`, `/actuator/beans` and `/actuator/mappings` all return 404 at runtime (T27, T32).
- Observability hooks exist: Micrometer `@Timed` on handlers, a custom `domain_events` counter, `http.server.requests` with URI/status tags, Prometheus registry, and a bundled Prometheus + Grafana compose stack (monitoring/).
- Book state changes use optimistic-locking predicates (`WHERE version = ?`) and the read-model insert handlers treat `DuplicateKeyException` as an idempotent no-op, which is a sound pattern for at-least-once event delivery.
- HTTP semantics for framework-handled cases are correct: 405 for unsupported methods (T22), 415 for unsupported media types (T23), 404 for unknown holds/checkouts (T12, T35), 204 for successful DELETE, HAL/HAL-FORMS affordances describe the allowed `delete` action on each hold.
- No application credentials exist in code or config: the only datastore is an in-memory H2 created by `EmbeddedDatabaseBuilder`, and the seed data runner is gated to the `local` profile (`LendingDatabaseConfig:62-64`).
- Domain invariants are enforced inside the model (`HoldDuration` rejects non-positive days, `@NonNull` on all value objects, `Either<BookHoldFailed, ...>` for business rejections), and a broad Spock test suite with CircleCI + codecov is present - a good base for contract tests.
## Needs confirmation

- The orchestrator's canonical commands (`mvn compile --settings /tmp/empty-settings.xml`) could not complete on this VM because repo1.maven.org returned HTTP 429 to direct downloads; compile/test/run succeeded only via a temporary local mirror settings file (not committed). Re-run the build in CI with normal registry access to confirm nothing differs.
- `alex-baker-cdm/library_demo` is reported by the orchestrator as byte-identical; I did not clone it. Confirm the hash and apply all LIB-* findings to that repo as well.
- Dependency advisory counts came from the resolved `mvn dependency:list` fed through the OSV API because osv-scanner's Maven Central lookups returned HTTP 429; the critical/high totals and fix versions should be re-confirmed with osv-scanner or OWASP dependency-check in CI.
- Checkout and return flows (BookCheckedOut/BookReturned) have no REST endpoint, so the `/checkouts` read side could only be exercised as empty collections; the partial-failure behaviour of those multi-step writes was not verified.
- CORS pre-flight returned 400 (T24); confirm whether an upstream gateway is expected to handle CORS and security headers in production, in which case LIB-F-03 reduces to documentation.
- The Grafana default password (LIB-F-02) is in a local docker-compose monitoring stack; confirm whether this compose file is ever used outside developer machines before treating it as a production secret exposure.
- No API specification was found in the repo; confirm with the platform team that none is maintained externally (e.g. in an API portal) before counting this repo as spec-less in the aggregate.
## Appendix: commands run and tool versions

| Tool | Version |
|---|---|
| Java | OpenJDK 11.0.32.1 (`/usr/lib/jvm/java-11-openjdk-amd64`) |
| Maven | Apache Maven 3.6.3 |
| curl | 7.81.0 |
| osv-scanner | 2.6.0 (package resolution via api.osv.dev because Maven Central returned HTTP 429) |

```bash
git clone git@github.com:alex-baker-cdm/library_java.git && git checkout master
# temporary, local-only (restored before commit): move pre-existing non-compiling sources out of the tree
mv src/main/java/io/pillopl/library/ui /tmp/excluded/ui
mv src/main/java/io/pillopl/library/lending/book/new_model /tmp/excluded/new_model
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
mvn compile --settings /tmp/mirror-settings.xml          # orchestrator's /tmp/empty-settings.xml -> HTTP 429 from repo1.maven.org
mvn test    --settings /tmp/mirror-settings.xml          # 89 Spock tests run, 0 failures, 0 errors, 0 skipped
SPRING_PROFILES_ACTIVE=local mvn spring-boot:run --settings /tmp/mirror-settings.xml > /tmp/app.log 2>&1 &
# seeded ids are logged by LendingDatabaseConfig (Created bookId / libraryBranchId / patronId)
curl -s -w 'HTTP %{http_code}\n' http://localhost:8080/profiles/<patronId>                     # T01 -> 200, no auth
curl -s -w 'HTTP %{http_code}\n' http://localhost:8080/profiles/11111111-1111-1111-1111-111111111111   # T08 -> 200 (IDOR)
curl -s -w 'HTTP %{http_code}\n' -X POST -H 'Content-Type: application/json' \
     -d '{"bookId":"<bookId>","libraryBranchId":"<branchId>","numberOfDays":3}' http://localhost:8080/profiles/<patronId>/holds   # T04 -> 200
curl -s -w 'HTTP %{http_code}\n' -X DELETE http://localhost:8080/profiles/<patronId>/holds/<bookId>    # T10 -> 204 (hold still listed afterwards, T11)
curl -s -w 'HTTP %{http_code}\n' -X POST -H 'Content-Type: application/json' -d '{}' .../holds        # T15 -> 500 "libraryBranchId is marked @NonNull but is null"
curl -s -w 'HTTP %{http_code}\n' -X POST -H 'Content-Type: application/json' -d '{...,"numberOfDays":2147483647}' .../holds  # T18 -> 200, till "+5881637-04-15"
python3 -c "import json;json.dump({'bookId':'<bookId>','libraryBranchId':'<branchId>','numberOfDays':3,'pad':'x'*12000000},open('/tmp/big.json','w'))"
curl -s -w 'HTTP %{http_code}\n' -X POST -H 'Content-Type: application/json' --data-binary @/tmp/big.json .../holds   # T33 -> 500 after full parse
curl -s -w 'HTTP %{http_code}\n' http://localhost:8080/actuator/prometheus                       # T29 -> 200, no auth
mvn dependency:list -DincludeScope=runtime -DoutputFile=/tmp/deps.txt --settings /tmp/mirror-settings.xml
# 67 artifacts -> api.osv.dev/v1/querybatch + /v1/vulns/{id}: 183 advisories, 27 critical + 92 high with a fix
```
