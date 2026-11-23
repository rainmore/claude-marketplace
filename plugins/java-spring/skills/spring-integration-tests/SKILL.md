---
name: spring-integration-tests
description: Tests beyond plain unit tests for Spring apps — a separate Gradle integrationTest source set for Docker-backed tests, Testcontainers PostgreSQL started by hand, running the production *Sql constants and real migration scripts against it, standalone MockMvc controller tests, and pinning time with Clock.fixed. Use when testing native SQL, migrations, JSON columns, concurrency, controllers, or time-dependent logic. Avoid @SpringBootTest; build only the pieces the test needs.
---

# Integration and Slice Tests

Naming, AssertJ and Mockito rules from `java-unit-tests` apply here too. This skill covers what to
test against real infrastructure and how to keep it fast.

**Default: no `@SpringBootTest`.** Starting the whole context is slow and hides which beans a
test depends on. Build the object under test by hand, start only the infrastructure it touches.

## 1. Choosing the kind of test

| Code under test | Test with | Source set |
| --- | --- | --- |
| Service / domain logic | Mockito unit test | `src/test` |
| Controller (routing, model, view, status, permissions) | standalone MockMvc | `src/test` |
| Native SQL, `SKIP LOCKED`, `RETURNING`, JSON columns, migrations | Testcontainers PostgreSQL | `src/integrationTest` |
| Time-dependent logic | `Clock.fixed` injected | wherever the logic lives |

Never test PostgreSQL-specific SQL against H2. It passes there and fails in production.

## 2. The `integrationTest` source set (Gradle)

Docker-backed tests live apart so `test`, `check` and `build` stay Docker-free:

```groovy
sourceSets {
    integrationTest {
        compileClasspath += sourceSets.main.output
        runtimeClasspath += sourceSets.main.output
    }
}
configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
    integrationTestCompileOnly.extendsFrom testCompileOnly
    integrationTestAnnotationProcessor.extendsFrom testAnnotationProcessor
}
tasks.register('integrationTest', Test) {
    description = 'Runs the tests in src/integrationTest (needs Docker).'
    group = 'verification'
    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath
    shouldRunAfter tasks.named('test')
}
dependencies {
    integrationTestImplementation 'org.testcontainers:testcontainers-postgresql'
    integrationTestImplementation 'org.testcontainers:testcontainers-junit-jupiter'
}
```

Run with `./gradlew :<module>:integrationTest`. Put a test here **only** if it needs Docker.

## 3. Testing native SQL against PostgreSQL

Run the exact production string from the `*Sql` class on the schema built by the real migration.
Put the test in the **same package** as the `*Sql` class, so the package-private constants are visible:

```java
/** Runs the delivery queue SQL against PostgreSQL, on the schema from the real db/021 migration. */
class InvoiceDeliveryQueueSqlTest {

    private static final LocalDateTime NOW = LocalDateTime.of(2026, 11, 20, 13, 0);

    private static PostgreSQLContainer postgres;
    private static Connection connection;
    private static NamedParameterJdbcTemplate jdbc;

    @BeforeAll
    static void startDatabase() throws Exception {
        postgres = new PostgreSQLContainer("postgres:16-alpine");
        postgres.start();
        connection = openConnection();
        jdbc = new NamedParameterJdbcTemplate(new SingleConnectionDataSource(connection, true));

        // Minimal stand-ins for tables the migration references, then the migration itself
        jdbc.getJdbcTemplate().execute("CREATE TABLE invoice (id BIGSERIAL PRIMARY KEY);");
        jdbc.getJdbcTemplate().execute(Files.readString(Path.of("../db/021_invoice_delivery_queue.sql")));
    }

    @AfterAll
    static void stopDatabase() throws SQLException {
        connection.close();
        postgres.stop();
    }

    @BeforeEach
    void clean() {
        jdbc.getJdbcTemplate().execute("DELETE FROM invoice_delivery_queue;");
    }

    @Test
    void claimDue_shouldClaimTheOldestDueUpToTheLimit_andMarkThemProcessing() {
        var first = insert(NOW.minusMinutes(30));
        var second = insert(NOW.minusMinutes(20));
        var third = insert(NOW.minusMinutes(10));

        var claimed = jdbc.queryForList(InvoiceDeliveryQueueSql.CLAIM_DUE, Map.of("now", NOW, "size", 2), Long.class);

        assertThat(claimed).containsExactlyInAnyOrder(first, second);
        assertThat(statusOf(third)).isEqualTo("PENDING");
    }
}
```

- Start the container in `@BeforeAll` and stop it in `@AfterAll` — one per class, not per test.
  Clean tables in `@BeforeEach`.
- Pin the image tag to the production major version (`postgres:16-alpine`).
- Load the **real migration file** rather than re-typing DDL, so the test also proves the migration
  runs. Create only minimal stand-ins for tables it references.
- Use a fixed `NOW` constant and pass it as a parameter; never rely on the database's `now()` in
  assertions.

### Concurrency

For `SKIP LOCKED`, open two connections with auto-commit off, claim in both before committing
either, and assert the claimed id sets are disjoint and together cover all rows:

```java
@Test
void claimDue_shouldGiveTwoConcurrentClaimsDisjointRows() throws Exception {
    insertMany(4);
    try (var tx1 = openConnection(); var tx2 = openConnection()) {
        tx1.setAutoCommit(false);
        tx2.setAutoCommit(false);

        var a = claim(tx1, 2);
        var b = claim(tx2, 2);

        assertThat(a).doesNotContainAnyElementsOf(b);
        assertThat(a.size() + b.size()).isEqualTo(4);
    }
}
```

### Hibernate mappings (JSON columns, converters)

Build a bare `SessionFactory` over the container for just the entity under test, persist, reload,
and also read the raw column with a native query to assert the stored format:

```java
sessionFactory = new Configuration()
    .addAnnotatedClass(EmailListHolder.class)
    .setProperty("hibernate.connection.url", postgres.getJdbcUrl())
    .setProperty("hibernate.connection.username", postgres.getUsername())
    .setProperty("hibernate.connection.password", postgres.getPassword())
    .setProperty("hibernate.hbm2ddl.auto", "none")
    .buildSessionFactory();
```

### Data migrations

For a script that converts existing data, insert rows in the **old** format, run the script, assert
the new format — then run it a second time and assert nothing changed (scripts must be safe to rerun).

## 4. Controller tests with standalone MockMvc

```java
class InvoiceViewControllerTest {

    private InvoiceService invoiceService;
    private InvoiceViewController controller;

    @BeforeEach
    void setUp() {
        invoiceService = mock(InvoiceService.class);
        controller = new InvoiceViewController(invoiceService);
    }

    @Test
    void list_shouldReturnForbidden_whenUserLacksTheAuthority() throws Exception {
        mockMvc(controller, userWith())
            .perform(get("/invoices"))
            .andExpect(status().isForbidden())
            .andExpect(view().name(View.DEFAULT_ERROR_VIEW_PATH));
    }
}
```

- `MockMvcBuilders.standaloneSetup(controller)` with a view resolver — no Spring context. Put the
  builder and a `userWith(authorities...)` principal helper in a shared test-support class.
- Use literal paths (`get("/invoices")`). The URL is the public contract under test, and the
  controller's `URI_*` constants are private.
- For every protected endpoint, test both the allowed and the forbidden path.
- For validated forms, stub the validator (`when(validator.supports(any())).thenReturn(true)`) and
  use `doAnswer` to add errors when testing the re-render path.

## 5. Pinning time

Production code takes a `Clock`; tests pass a fixed one:

```java
private static final Instant NOW = Instant.parse("2026-11-20T02:00:00Z");
private final Clock clock = Clock.fixed(NOW, ZoneId.of("Australia/Sydney"));

@BeforeEach
void setUp() {
    service = new ReminderService(repository, clock);
}
```

Use the application's real zone in the fixed clock so date boundaries match production. Code that
calls `LocalDateTime.now()` directly can't be tested this way — change it to `now(clock)`.

## 6. Files

Use `@TempDir Path dir` for anything a test writes to disk; never write to the project directory or
a fixed `/tmp` path.

## Checklist

- [ ] No `@SpringBootTest`; only the needed pieces are constructed
- [ ] PostgreSQL-specific SQL tested on Testcontainers in `src/integrationTest`, never H2
- [ ] Production `*Sql` constants and real migration files are what the test executes
- [ ] `SKIP LOCKED` claims have a two-connection concurrency test
- [ ] Controllers tested with standalone MockMvc, allowed and forbidden paths
- [ ] Time pinned with `Clock.fixed` in the application zone
