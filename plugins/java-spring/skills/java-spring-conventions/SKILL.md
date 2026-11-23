---
name: java-spring-conventions
description: Java and Spring Boot coding conventions, patterns, and best practices for enterprise applications. Apply when writing or reviewing Java code, Spring components, MapStruct mappers, Temporal workflows, or unit tests.
---

# Java & Spring Boot Coding Conventions

Apply these conventions when writing Java code for Spring Boot enterprise applications.

## 1. Class Structure & Annotations

### Component Annotations
- **Always use constructor injection** with `@RequiredArgsConstructor` and `final` fields
- Use appropriate Spring stereotypes:
  - `@Component` for general Spring-managed beans
  - `@Service` for business logic services
  - `@RestController` or `@Controller` for web endpoints
  - `@Repository` for data access (if not using Spring Data)

### Lombok Annotations (Standard Pattern)
```java
@Slf4j                    // For logging (use log.info(), log.error(), etc.)
@Component                // Or @Service, @RestController, etc.
@RequiredArgsConstructor  // Constructor injection for final fields
public class MyService {

    private final DependencyOne dependencyOne;
    private final DependencyTwo dependencyTwo;

    // Methods here
}
```

**Never use:**
- `@Autowired` on fields (prefer constructor injection)
- `@Data` on entities or DTOs with complex logic (use specific annotations instead)

### MapStruct Mappers
```java
@Mapper(componentModel = ComponentModel.SPRING)
public interface MyMapper {
    @Mapping(target = "targetField", source = "sourceField")
    @Mapping(target = "complexField", source = "source", qualifiedByName = "mapComplexField")
    TargetDto toTargetDto(SourceDto source);

    @Named("mapComplexField")
    default ComplexType mapComplexField(SourceType source) {
        // Custom mapping logic
        return result;
    }
}
```

**Conventions:**
- Use `componentModel = ComponentModel.SPRING` to enable dependency injection
- No `INSTANCE` field: inject the mapper; tests use `Mappers.getMapper(...)`
- Use `@Named` for custom mapping methods
- Prefix custom mapping method names with qualifier name (e.g., "map", "convert", "transform")
- Use `qualifiedByName` to reference custom mapping methods

## 2. Validation

### Parameter Validation
- Use Jakarta validation annotations on method parameters:
  - `@NotNull` for required parameters
  - `@Valid` for nested object validation
  - Consider: `@NotBlank`, `@NotEmpty`, `@Size`, `@Min`, `@Max`, `@Pattern`

```java
public PartyData processData(
        @NotNull final RequestHeaderData requestHeaderData,
        @NotNull final PartyData partyData,
        @Valid final CreateAccountReqType createAccountReqType
) {
    // Method implementation
}
```

### Field Immutability
- Use `final` for:
  - Injected dependencies (always)
  - Method parameters when appropriate for clarity
  - Local variables that won't be reassigned

## 3. Logging

### SLF4J with Lombok
```java
@Slf4j
public class MyService {

    public void processRequest(String requestId) {
        log.info("Processing request: {}", requestId);

        try {
            // Business logic
            log.debug("Request {} processed successfully", requestId);
        } catch (Exception e) {
            log.error("Failed to process request: {}", requestId, e);
            throw e;
        }
    }
}
```

**Best practices:**
- Use parameterized logging: `log.info("Message: {}", value)` not string concatenation
- Log at appropriate levels:
  - `ERROR`: Exceptions, failures requiring attention
  - `WARN`: Unexpected but handled situations
  - `INFO`: Important business events, lifecycle events
  - `DEBUG`: Detailed diagnostic information
- Include exception as last parameter: `log.error("Message", exception)`
- Never log sensitive data (passwords, tokens, PII)

## 4. Exception Handling

### Specific Exceptions
- **Never catch generic `Exception`** unless at application boundary (controllers, workers)
- Create specific exception types for business errors
- Let Spring handle exception translation when possible

```java
// Good
try {
    repository.save(entity);
} catch (DataAccessException e) {
    log.error("Failed to save entity: {}", entity.getId(), e);
    throw new EntityPersistenceException("Could not save entity", e);
}

// Avoid
try {
    repository.save(entity);
} catch (Exception e) {  // Too broad
    // ...
}
```

### Exception Propagation
- Throw exceptions for exceptional conditions
- Use custom exceptions that extend `RuntimeException` for business errors
- Include context in exception messages
- Always include the cause when wrapping exceptions

## 5. Null Safety

### Use Optional for Potential Absence
```java
// Return Optional when value may not exist
public Optional<User> findUserById(String id) {
    return userRepository.findById(id);
}

// Use it
userService.findUserById(id)
    .map(this::transform)
    .orElseThrow(() -> new UserNotFoundException(id));
```

### Default Methods in Interfaces/Mappers
```java
@Named("mapAccountSubType")
default CMAAccountSubType mapAccountSubType(CreateAccountReqTypeTrustDetails trustDetails) {
    if (trustDetails == null) {
        return null;  // Explicit null handling
    }
    return TrustTypeEnum.SMSF.equals(trustDetails.getTrustType())
        ? CMAAccountSubType.SMSF
        : null;
}
```

**Conventions:**
- Always check for null when it's possible
- Use `Optional` for return values that may be absent
- Use `@NotNull` validation to prevent null parameters
- Consider using `Objects.requireNonNull()` for critical null checks

## 6. Temporal Workflow Activities

### Activity Implementation
```java
@Slf4j
@Component
@RequiredArgsConstructor
@ActivityImpl(taskQueues = "${config.path.to.task-queue-name}")
public class MyActivityImpl implements MyActivity {

    private final ServiceDependency service;

    @Override
    public ResultType performActivity(
            @NotNull final InputType input
    ) {
        log.info("Starting activity for: {}", input.getId());

        // Activity logic
        var result = service.process(input);

        log.info("Activity completed for: {}", input.getId());
        return result;
    }
}
```

**Conventions:**
- Implement activity interfaces
- Use externalized configuration for task queue names
- Log activity start/completion
- Keep activities idempotent when possible
- Use `@NotNull` validation on parameters

## 7. Testing Best Practices

### Test Structure
```java
@ExtendWith(MockitoExtension.class)
class MyServiceTest {

    @Mock
    private DependencyOne dependencyOne;

    @Mock
    private DependencyTwo dependencyTwo;

    @Captor
    private ArgumentCaptor<SomeType> argumentCaptor;

    private MyService service;

    private static final String TEST_CONSTANT = "test-value";

    @BeforeEach
    void setUp() {
        service = new MyService(dependencyOne, dependencyTwo);
    }

    @Test
    void methodName_shouldExpectedBehavior_whenCondition() {
        // Given
        var input = createTestInput();
        when(dependencyOne.someMethod(any())).thenReturn(expectedValue);

        // When
        var result = service.methodName(input);

        // Then
        assertThat(result)
            .isNotNull()
            .satisfies(r -> {
                assertThat(r.getField()).isEqualTo(expectedValue);
            });

        verify(dependencyOne, times(1)).someMethod(any());
    }
}
```

### Testing Conventions

**Test Method Naming:**
- Pattern: `methodName_shouldExpectedBehavior_whenCondition`
- Examples:
  - `publishInProgressStatusMessage_shouldBuildEventAndPublish_whenInProgressStatus`
  - `mapAccountSubType_shouldReturnSMSF_whenTrustTypeIsSMSF`
  - `findUser_shouldThrowException_whenUserNotFound`
- **Do not add `@DisplayName`.** A method name following this pattern is
  already the explanation; a separate display string is redundant and drifts
  out of sync with the method name over time. Existing `@DisplayName` is legacy:
  remove it from any test file you edit. Put the effort into a name
  that's descriptive on its own instead of adding an annotation on top of it.
  ```java
  // ❌ Don't
  @Test
  @DisplayName("should return the expected result when the input is valid")
  void methodName_shouldReturnExpectedResult_whenValidInput() { ... }

  // ✅ Do
  @Test
  void methodName_shouldReturnExpectedResult_whenValidInput() { ... }
  ```

**Test Organization:**
- `// Given` / `// When` / `// Then` comments are **optional**. Use them when a test's setup is
  long enough that the phases aren't obvious; leave them out when blank lines already separate
  arrange, act and assert. Never write an empty `// Given` just to have the set.
- Keep test data setup in `@BeforeEach` when shared across tests
- Use static constants for test data values
- One logical assertion per test (but can verify multiple aspects)

**Mocking:**
- Use `@Mock` for dependencies
- Use `@Captor` for argument capture
- Use `when(...).thenReturn(...)` for stubbing
- Use `verify(...)` to assert interactions
- Use `times(n)` to verify exact call count
- Use `any()` matchers when exact values don't matter
- Use `inOrder()` to verify method call order

**Assertions (AssertJ):**
```java
// Basic assertions
assertThat(result).isNotNull();
assertThat(result).isEqualTo(expected);
assertThat(result).isSameAs(sameObject);

// Fluent chaining
assertThat(result)
    .isNotNull()
    .satisfies(r -> {
        assertThat(r.getField()).isEqualTo(value);
        assertThat(r.getOther()).isNotEmpty();
    });

// Exception assertions
assertThatThrownBy(() -> service.method())
    .isInstanceOf(ExpectedException.class)
    .hasMessage("Expected message")
    .hasCauseInstanceOf(RootCause.class);

// Collection assertions
assertThat(list)
    .isNotEmpty()
    .hasSize(3)
    .contains(element)
    .extracting(Item::getName)
    .containsExactly("name1", "name2", "name3");
```

**Test Coverage:**
- Happy path (successful execution)
- Edge cases (boundary conditions, empty inputs, etc.)
- Error cases (exceptions, validation failures)
- Null handling
- Multiple invocations/state transitions
- Order of operations (`inOrder()` verification)
- Argument values passed to dependencies

## 8. Code Organization

**Project conventions win.** If the repo has a naming skill (e.g. `<project>-naming-conventions`)
or a naming section in `CLAUDE.md`, follow it over everything in this section. Otherwise, open the
nearest existing feature and copy its package layout and suffixes before inventing new ones.

### Package Structure
Prefer feature-first, with layers inside each feature. **Package names are a single singular
term**: `controller`, `service`, `dto`, `util`, never `controllers`, `services`, `utils`.
```
com.example.app
├── config/                  # @Configuration, *Properties
└── <feature>/
    ├── controller/
    └── service/             # *Service, *Repository (+Custom/Impl/Sql), *Specification
        └── dto/             # DTOs, mappers, validators for this feature
```
Repositories live in the `service` package beside the services that use them, not in a separate
`repository` package.

### Naming Conventions
- **Classes:** one suffix per role, used consistently — `*Controller`, `*Service`, `*Repository`,
  `*Mapper`, `*Dto`, `*Config`, `*Properties`, `*Exception`. Entities and value `record`s take no suffix.

- **Methods:**
  - Lookups: `find*` returns a collection or page; `findOne*` returns a single item (throws
    not-found, or `Optional` when absence is normal)
  - `get*` / `set*` are **only** property accessors. A method that queries, loads or computes is
    never `get*`: use `findOne*`, `build*`, `count*`, `resolve*`
  - Other verb prefixes: `save*`, `create*`, `update*`, `archive*`, `build*`, `publish*`
  - Entity state changes: `mark<State>()` (`markCompleted`), never `markAs<State>`
  - Boolean methods: `is*`, `has*`, `can*`, `should*`
  - Converters/mappers: `to*`, `from*`, `map*`

- **Variables:**
  - Use descriptive names, avoid abbreviations unless common (e.g., `dto`, `id`, `aml`)
  - Use `var` for local variables when type is obvious from right-hand side
  - Collections: plural names (`users`, `accounts`, `details`)

## 9. Modern Java Patterns

### Records vs POJOs

Default to `record` for new types: `@ConfigurationProperties`, SQL row and projection results,
multi-value results, read-only view and REST DTOs, events, keys and value types. Use a Lombok POJO
only for JPA entities, form-backing DTOs, MapStruct `@MappingTarget` targets, step-built or
many-optional-field objects, and types that must extend a class. The full decision table is in
`spring-boot-configuration` §4.

### Use `var` for Local Variables
```java
// Good - type is obvious from assignment
var user = userRepository.findById(id);
var result = service.processRequest(request);

// Good - use var for non-reassignable variables (instead of explicit type)
var container = GateInGateOutContainerDto.builder()
    .containerNumber("ABC123")
    .containerType("40HQ")
    .build();

var carrier = GateOutCarrierDto.builder()
    .idOfTheMeansOfTransport("VESSEL_NAME")
    .build();

// Avoid - type is not obvious
var data = getData();  // What type is this?

// Never use for fields or parameters
```

**When to use `var`:**
- Local variables where the type is obvious from the right-hand side
- Non-reassignable variables (single assignment)
- Builder patterns and constructor calls
- Method calls where return type is explicit

**When NOT to use `var`:**
- Fields (class/instance variables) - always use explicit type
- Method parameters - always use explicit type
- Return types - always use explicit type

### Stream API
```java
// Use streams for collection operations
var activeUsers = users.stream()
    .filter(User::isActive)
    .map(this::toDto)
    .collect(Collectors.toList());

// Use method references when possible
.map(User::getName)          // Not: .map(u -> u.getName())
.filter(Objects::nonNull)     // Not: .filter(obj -> obj != null)
```

### Switch Expressions (Java 14+)
```java
// Modern switch expression
return switch (status) {
    case PENDING -> "Processing";
    case COMPLETED -> "Done";
    case FAILED -> "Error";
    default -> "Unknown";
};
```

## 10. Common Pitfalls to Avoid

- **Don't** use field injection (`@Autowired` on fields)
- **Don't** catch generic `Exception` unless at application boundary
- **Don't** return null - use `Optional` or empty collections
- **Don't** use `@Data` on entities (can cause issues with lazy loading, circular references)
- **Don't** log and rethrow exceptions (creates duplicate logs)
- **Don't** swallow exceptions without logging
- **Don't** use string concatenation for logging (`log.info("Value: " + value)`)
- **Don't** create utility classes (`*Util`, `*Helper`, `@UtilityClass`) or public constants. See `java-code-placement` for where the logic or value goes instead
- **Don't** use Hungarian notation or prefixes (`strName`, `mField`)

## 11. Documentation

### JavaDoc
Add JavaDoc for:
- Public APIs and interfaces
- Complex business logic
- Non-obvious behavior or side effects

```java
/**
 * Publishes a digital application event and updates the application status.
 *
 * @param requestHeaderData request context and correlation data
 * @param partyData party information including account request details
 * @param applicationStatus the status to publish and set on the account request
 * @return the updated PartyData with new application status
 * @throws DigitalEventPublishException if event publication fails
 */
public PartyData publishInProgressStatusMessage(
        @NotNull RequestHeaderData requestHeaderData,
        @NotNull PartyData partyData,
        @NotNull ApplicationStatus applicationStatus
) {
    // Implementation
}
```

### Comments
- Prefer self-documenting code over comments
- Use comments to explain **why**, not **what**
- Keep comments updated with code changes
- Remove commented-out code (use version control instead)

---

## Summary Checklist

When writing Java/Spring Boot code, ensure:

- [ ] Using constructor injection with `@RequiredArgsConstructor` and `final` fields
- [ ] Applied appropriate Spring stereotype annotations
- [ ] Added `@Slf4j` for logging
- [ ] Using parameterized logging (not string concatenation)
- [ ] Validated method parameters with `@NotNull`/`@Valid`
- [ ] Handled null cases explicitly or used `Optional`
- [ ] Used specific exception types (not generic `Exception`)
- [ ] **Used `var` for all non-reassignable local variables** (essential)
- [ ] Named tests with pattern: `method_shouldBehavior_whenCondition`
- [ ] **No `@DisplayName` annotations** — the method name is the explanation
- [ ] Arrange / act / assert separated (Given/When/Then comments optional)
- [ ] Used AssertJ for assertions (`assertThat`)
- [ ] Verified mock interactions with appropriate matchers
- [ ] Covered happy path, edge cases, and error scenarios in tests
- [ ] Followed package and naming conventions
- [ ] Used MapStruct with `componentModel = SPRING` for mappings
- [ ] Made Temporal activities idempotent when possible

Apply these patterns consistently for maintainable, testable, and idiomatic Spring Boot code.