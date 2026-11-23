---
name: spring-boot-configuration
description: Spring Boot configuration, time handling and records — @ConfigurationProperties as records with nested groups, @DefaultValue and Duration fields, when to use a record vs a Lombok POJO, environment-variable defaults in application.yml, documenting every property, an injected Clock in the application time zone, and plain SQL migration scripts. Use when adding a setting, a scheduled interval or timeout, a new DTO/value/result type, anything that reads the current time, or a database schema change. Never call LocalDateTime.now() without a Clock in code that has one injected.
---

# Configuration, Time and Migrations

## 1. `@ConfigurationProperties`

**Properties are records.** One record per prefix, nested records for groups. Records bind through
their constructor, so the settings are immutable after startup and nothing can change them at
runtime by calling a setter.

```java
/**
 * Configuration of the invoice delivery batch job.
 *
 * @param batch batch sizing and timing
 */
@Validated
@ConfigurationProperties(prefix = "invoice-delivery")
public record InvoiceDeliveryProperties(@DefaultValue @Valid Batch batch) {

    /**
     * @param fixedDelay   wait between batches, measured from the end of the previous batch
     * @param size         most rows processed in one batch
     * @param staleTimeout a row in progress longer than this was abandoned by a stopped worker
     */
    public record Batch(
        @DefaultValue("2m") Duration fixedDelay,
        @DefaultValue("5") @Positive int size,
        @DefaultValue("10m") Duration staleTimeout
    ) {
    }
}
```

Register them once, on the application class or a config class:

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class Application { ... }
```

- **No `@Component` on a properties record.** Register it with `@ConfigurationPropertiesScan` (or
  `@EnableConfigurationProperties(InvoiceDeliveryProperties.class)`). A `@Component` record is
  created as an ordinary bean, and its constructor is never bound to configuration.
- **Defaults go in `@DefaultValue`**, not in field initialisers (a record has none). Put a bare
  `@DefaultValue` on a nested record parameter, so the group is created with its own defaults even
  when the YAML leaves it out. Without that the nested record is `null`.
- Validate with `@Validated` on the record and constraints on its parameters (`@Positive`,
  `@NotBlank`), plus `@Valid` on nested groups. Invalid configuration then stops startup.
- Document every component with `@param` in the record's Javadoc.
- Collections: a `List<String>` component binds as an immutable list; don't try to modify it.
- Tests build the record directly: `new InvoiceDeliveryProperties(new Batch(Duration.ofSeconds(1), 2, Duration.ofMinutes(1)))`.
- Use `Duration` for every interval and timeout (`2m`, `30s`, `10m` in YAML), never `long millis`.
- Inject the properties bean; don't scatter `@Value("${...}")` across services.
- Include `spring-boot-configuration-processor` as an `annotationProcessor`.
- Reference a property from an annotation with a placeholder:
  `@Scheduled(fixedDelayString = "${invoice-delivery.batch.fixed-delay}")`.

## 2. `application.yml`

Settings that differ per deployment read an environment variable with a default:

```yaml
invoice-delivery:
  batch:
    fixed-delay: ${APP_INVOICE_DELIVERY_BATCH_DELAY:2m}
    size: ${APP_INVOICE_DELIVERY_BATCH_SIZE:5}
    stale-timeout: 10m
```

- Profiles: `application.yml` (shared), `application-local.yml`, `application-dev.yml`, and the
  test profile set by the build (`spring.profiles.active=test`).
- Never commit real passwords, tokens or keys in any profile file. Use an environment variable
  with no default.
- When you add or change a property, update the project's configuration reference (README table:
  property, default, meaning) in the same change.

## 3. Time

There is exactly one application time zone, from configuration, pinned at startup:

```java
@Configuration
public class ClockConfig {

    @Bean
    public Clock clock(AppProperties appProperties) {
        return Clock.system(ZoneId.of(appProperties.getTimeZone()));
    }
}
```

- Inject `Clock` and call `LocalDateTime.now(clock)` / `LocalDate.now(clock)`. Direct
  `LocalDateTime.now()` makes the code untestable for time even when the zone is right.
- Store wall-clock `LocalDateTime` in the application zone consistently; convert to another zone
  (`atZone(appZone).withZoneSameInstant(portZone)`) only at the edge that needs it.
- Measure elapsed time between two `LocalDateTime`s via `atZone(zone)` and `Duration.between`, so
  DST changes are counted correctly.
- Changing the application zone on an existing database shifts every stored time. Treat it as a
  data migration, not a config change.
- Tests: `Clock.fixed(instant, appZone)` — see `spring-integration-tests`.

## 4. Record or POJO

**Default to a record.** Use a POJO (a Lombok class) only when a framework or the object's
lifecycle needs mutation, a no-argument constructor, or inheritance.

### Use a `record` for

| Kind | Example |
| --- | --- |
| `@ConfigurationProperties` (section 1) | `InvoiceDeliveryProperties`, nested `Batch` |
| Rows from native SQL or projections | `ClaimedInvoice(long id, LocalDateTime dueAt)` |
| Method results carrying more than one value | `ResetResult(int reset, int skipped)`, `RerunResult` |
| Read-only view DTOs, built once and rendered | `UserAccountNameDto(long id, String name)`, label and summary DTOs |
| REST request and response bodies (Jackson binds records) | `InvoiceSummaryResponse` |
| Events, queue messages, channel payloads | `InvoiceDelivered(long invoiceId, Instant at)` |
| Identity objects: security principal, authority | `CurrentUser`, `GrantedAuthorityImpl` |
| Small value types with rules | `EmailAddressList`, `ReportDateRange(LocalDate from, LocalDate to)` |
| Map or cache keys, `Set` elements (needs value equality) | `ContainerBookingKey(String container, String booking)` |
| Test data in parameterized tests | `@MethodSource` argument holders |

### Use a POJO (Lombok class) for

| Kind | Why a record can't do it |
| --- | --- |
| JPA `@Entity` | Hibernate needs a no-arg constructor, mutable fields and subclass proxies for lazy loading; records are final and immutable |
| `@MappedSuperclass` / auditable base models | Records can't extend a class |
| Form-backing objects (`th:object`, `@ModelAttribute` + `BindingResult`) | The form is re-rendered with partial, invalid input; binding nested paths (`priceAmount.total`) and defaults from `buildEmpty()` rely on setters |
| MapStruct `@MappingTarget` update methods | MapStruct writes into the existing object through setters |
| Objects built up in steps or with many optional fields | Use `@Builder` (`toBuilder = true`). A record with 10+ components is hard to read at the call site |
| Objects whose state changes over their life | e.g. a queue row moving `PENDING → PROCESSING`; use `mark<State>()` methods |
| Framework types that require a superclass | `AbstractMessageSource`, Selenium page objects |

Lombok for POJOs: `@Getter`/`@Setter` on entities (never `@Data`); `@Data @NoArgsConstructor
@AllArgsConstructor @Builder(toBuilder = true)` on form DTOs.

### Writing records

```java
public record ReportDateRange(LocalDate from, LocalDate to) {

    public ReportDateRange {                               // compact constructor: validate
        Objects.requireNonNull(from, "from");
        if (to != null && to.isBefore(from)) {
            throw new IllegalArgumentException("to %s is before from %s".formatted(to, from));
        }
    }

    public static ReportDateRange ofMonth(YearMonth month) { // named factories for common shapes
        return new ReportDateRange(month.atDay(1), month.atEndOfMonth());
    }

    public boolean contains(LocalDate date) {              // behaviour on the value is fine
        return !date.isBefore(from) && (to == null || date.isBefore(to));
    }
}
```

- Check required components and invariants in the compact constructor, so an invalid value can
  never exist.
- Copy collection components defensively: `emails = List.copyOf(emails);`. A record is only as
  immutable as its components; avoid arrays.
- No Lombok on records. `@Builder` on a record is a sign it should be a POJO, or should be split.
- Add `static of(...)` / `from(...)` factories for common ways to build it, e.g.
  `RobotHealthDto.of(health, now)`.
- MapStruct maps **into** records through their constructor; give it `to<Record>` methods as usual.
- Accessors are `name()`, not `getName()`. In Thymeleaf, `${dto.name}` works for both kinds.

### Changing an existing class

Don't convert existing POJOs to records as a side effect of other work. Callers that use setters
or `getX()` break. Use records for new types, and convert old ones only when asked.

## 5. SQL migrations

When the project manages schema with numbered scripts (no Flyway/Liquibase):

- Next number, descriptive name: `db/021_invoice_delivery_queue.sql`.
- Make every script **safe to run twice**: `CREATE TABLE IF NOT EXISTS`, `ADD COLUMN IF NOT
  EXISTS`, `CREATE INDEX IF NOT EXISTS`, `INSERT ... ON CONFLICT DO NOTHING`, data conversions
  guarded by a `WHERE` that only matches unconverted rows.
- Note in the change (and the README if operators run it) whether it must run **before** the new
  application version starts — e.g. when an entity's column mapping changes type.
- Cover non-trivial scripts with an integration test that runs the real file.

## Checklist

- [ ] New settings live in a `@ConfigurationProperties` **record**, registered by scan (no `@Component`), with `@DefaultValue` and `@param` Javadoc
- [ ] New value, result and read-only DTO types are records; entities, form DTOs and `@MappingTarget` types are POJOs
- [ ] Intervals and timeouts are `Duration`
- [ ] Deployment-specific values come from environment variables; no secrets committed
- [ ] README configuration reference updated
- [ ] Current time comes from the injected `Clock`
- [ ] Migration is numbered, rerunnable, and its run-order requirement is stated
