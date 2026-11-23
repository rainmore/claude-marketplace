---
name: spring-data-querydsl
description: Spring Data JPA + QueryDSL data-access conventions — repositories extending a QueryDSL-enabled base, query conditions kept in *Specification classes, soft delete by archiving, native SQL in *Sql constant classes behind *RepositoryCustom/*Impl, and JSON columns. Use when adding or changing an entity, repository, query, or native SQL in a project that uses QueryDSL. Never add JPQL @Query strings when a Specification condition will do.
---

# Spring Data JPA + QueryDSL

Data access goes through three pieces per aggregate. Before adding any, look for the project's own
base types (`BaseRepository`, `BaseSpecification`, auditable/archivable model superclasses) and
extend those rather than the raw Spring Data interfaces.

| Piece | Holds | Never holds |
| --- | --- | --- |
| `XRepository` | the interface Spring Data implements | query logic |
| `XSpecification` | `BooleanExpression` conditions and the default sort | persistence calls |
| `XService` | `@Transactional` boundaries, combining conditions, mapping to DTOs | `Q*` paths built inline when a Specification method exists |

All three, plus any `XRepositoryCustom` / `XRepositoryImpl` / `XSql`, live together in the
feature's `service` package. There is no separate `repository` package.

Service lookups: `find*` returns a collection or page, `findOne*` a single item, and `get*` is only
ever a property getter.

## 1. Repository

```java
@Repository
public interface InvoiceRepository extends BaseRepository<Invoice, Long> {
}
```

`BaseRepository` is a `@NoRepositoryBean` combining `JpaRepository` and
`QuerydslPredicateExecutor`, so every repository gets `findAll(Predicate, Pageable)`,
`findOne(Predicate)`, `exists(Predicate)` and `count(Predicate)` without declaring anything.

- Don't add derived query methods (`findBySlugAndArchivedAtIsNull...`). Put the condition in the
  Specification and call `findOne(condition)`.
- Don't add JPQL `@Query` strings. If QueryDSL can't express it, it is native SQL (section 4).

## 2. Specification

One `@Component` per entity, extending the base so every entity answers the same questions:

```java
@Component
public class InvoiceSpecification extends BaseSpecification<Invoice, Long> {

    private static final QInvoice INVOICE = QInvoice.invoice;

    @Override public EntityPath<Invoice> path() { return INVOICE; }
    @Override public OrderSpecifier<?> defaultSort() { return INVOICE.issuedOn.desc(); }
    @Override public BooleanExpression idCondition(Long id) { return INVOICE.id.eq(id); }

    @Override
    public BooleanExpression archivedCondition(boolean archived) {
        return archived ? INVOICE.archivedAt.isNotNull() : INVOICE.archivedAt.isNull();
    }

    public BooleanExpression customerCondition(long customerId) {
        return INVOICE.customer.id.eq(customerId);
    }

    /** Overlap of half-open ranges [from, to); a null `to` is open-ended. */
    public BooleanExpression overlapsCondition(LocalDate from, @Nullable LocalDate to) {
        var condition = INVOICE.effectiveTo.isNull().or(INVOICE.effectiveTo.after(from));
        return to == null ? condition : condition.and(INVOICE.effectiveFrom.before(to));
    }
}
```

- `actualCondition()` (= not archived) and `actualCondition(id)` come from the base. Start almost
  every read from `actualCondition()` so archived rows never leak into results.
- Name methods `<what>Condition`. Compose with `.and()` / `.or()` in the Specification, not in the
  service, when the combination has a business name.
- Write the rule as a comment when it is not obvious from the expression (interval overlap,
  inclusive vs. exclusive bounds).

## 3. Service usage

```java
@Transactional(readOnly = true)
public Page<InvoiceDto> find(Pageable pageable) {
    var sorted = pageable.getSort().isEmpty()
        ? QPageRequest.of(pageable.getPageNumber(), pageable.getPageSize(), invoiceSpecification.defaultSort())
        : pageable;
    return invoiceRepository.findAll(invoiceSpecification.actualCondition(), sorted).map(invoiceDtoMapper::toDto);
}

@Transactional(readOnly = true)
public InvoiceDto findOne(Long id) {
    return invoiceRepository.findOne(invoiceSpecification.actualCondition(id))
        .map(invoiceDtoMapper::toDto)
        .orElseThrow(() -> new EntityNotFoundException("Invoice %d not found".formatted(id)));
}
```

- Throw `EntityNotFoundException` (or the project's not-found type) for a missing id, never a bare
  `RuntimeException` — the web layer maps the specific type to 404; a `RuntimeException` becomes 500.
- Make the message name the entity you are actually looking up. Copy-pasted messages from another
  service are a common, misleading bug.
- `readOnly = true` on every read method.

## 4. Native SQL

When the query needs PostgreSQL features (`FOR UPDATE SKIP LOCKED`, `RETURNING`, `jsonb`, CTEs):

```java
/** Native SQL on {@code invoice}. Kept apart so integration tests run it as-is. */
final class InvoiceSql {

    static final String CLAIM_DUE = """
        UPDATE invoice SET status = 'PROCESSING', processing_started_at = :now
        WHERE id IN (
          SELECT id FROM invoice WHERE status = 'PENDING' AND due_at <= :now
          ORDER BY due_at, id LIMIT :size
          FOR UPDATE SKIP LOCKED)
        RETURNING id, due_at
        """;

    private InvoiceSql() {
    }
}
```

```java
public interface InvoiceRepositoryCustom {
    List<ClaimedInvoice> claimDue(LocalDateTime now, int size);
}

public class InvoiceRepositoryImpl implements InvoiceRepositoryCustom {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<ClaimedInvoice> claimDue(LocalDateTime now, int size) {
        List<?> rows = entityManager.createNativeQuery(InvoiceSql.CLAIM_DUE)
            .setParameter("now", now)
            .setParameter("size", size)
            .getResultList();
        return rows.stream()
            .map(row -> (Object[]) row)
            .map(row -> new ClaimedInvoice(((Number) row[0]).longValue(), toLocalDateTime(row[1])))
            .sorted(Comparator.comparing(ClaimedInvoice::dueAt))   // RETURNING does not keep order
            .toList();
    }
}
```

- SQL lives in a package-private `final` `*Sql` class of package-private `static final String`
  text blocks with named parameters. The `*RepositoryImpl` and the integration test sit in the same
  package, so both run exactly the same string without anything becoming `public` (see
  `spring-integration-tests` and `java-code-placement`).
- `InvoiceRepository extends BaseRepository<Invoice, Long>, InvoiceRepositoryCustom`.
- Map result rows into a `record`. Numbers come back as `Number` — convert with `.longValue()`,
  never cast to `Long`. Timestamps may arrive as `java.sql.Timestamp`.
- Short-circuit empty `IN (:ids)` collections before executing.

## 5. Entities

```java
@Getter
@Setter
@Entity
@Table(name = "invoice")
public class Invoice extends ArchivableAuditableModel {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    @NotEmpty
    @Size(max = 150)
    private String reference;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "JSON", nullable = false)
    private List<String> noticeEmails = new ArrayList<>();
}
```

- `@Getter` / `@Setter` on entities, **never `@Data`** (its `equals`/`hashCode`/`toString` walk lazy
  associations).
- Extend the auditable/archivable superclass instead of redeclaring `createdAt`/`archivedAt`.
- **Soft delete:** call `entity.archive(user)`; never `repository.delete(...)` a business record.
  Record edits with `entity.update(user)`.
- Mirror DB constraints with Bean Validation (`nullable = false` ↔ `@NotNull`/`@NotEmpty`,
  column length ↔ `@Size`).
- JSON list columns: `@JdbcTypeCode(SqlTypes.JSON)` with `JSON NOT NULL DEFAULT '[]'` in SQL, and
  initialise the field to an empty list. Prove the round-trip in an integration test.

## Checklist

- [ ] Repository extends the project's QueryDSL base; no derived or `@Query` methods added
- [ ] Every condition is a named method on the Specification
- [ ] Reads start from `actualCondition()`; list reads get `defaultSort()` when unsorted
- [ ] Missing id → not-found exception that names the right entity
- [ ] Native SQL in a `*Sql` class, called from `*RepositoryCustom`/`*Impl`, covered by an integration test
- [ ] Entity uses `@Getter`/`@Setter`, extends the auditable base, archives instead of deleting
