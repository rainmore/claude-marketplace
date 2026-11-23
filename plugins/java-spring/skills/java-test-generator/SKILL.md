---
name: java-test-generator
description: Generate comprehensive unit tests for Java classes following project testing patterns with Mockito and AssertJ. Use when asked to create tests, generate test cases, or add test coverage.
disable-model-invocation: false
---

# Java Test Generator

Generate comprehensive unit tests for Java classes following enterprise testing patterns.

## When to Use

- User asks to "generate tests", "create unit tests", "add test coverage"
- User requests "test this class", "write tests for..."
- After creating a new class or service that needs test coverage

## Test Generation Process

### 1. Analyze the Target Class

Before generating tests:
- Read the source class file
- Identify all public methods
- Understand dependencies (constructor parameters)
- Identify return types and parameters
- Look for validation annotations (`@NotNull`, `@Valid`)
- Check for exception handling

### 2. Test Class Structure

Generate tests following this structure:

```java
package com.macquarie.bfs.integration.trust.originations.workflow.[package];

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.MethodSource;
import org.junit.jupiter.params.provider.ValueSource;
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class [ClassName]Test {

    @Mock
    private DependencyOne dependencyOne;

    @Mock
    private DependencyTwo dependencyTwo;

    @Captor
    private ArgumentCaptor<SomeType> argumentCaptor;

    private [ClassName] classUnderTest;

    private static final String TEST_CONSTANT = "test-value";

    @BeforeEach
    void setUp() {
        classUnderTest = new [ClassName](dependencyOne, dependencyTwo);
    }

    // Test methods here
}
```

**Import notes for parameterized tests:**
- Add `org.junit.jupiter.params.ParameterizedTest`
- Add provider: `@ValueSource`, `@CsvSource`, or `@MethodSource`
- Only import the providers you use

### 3. Test Method Pattern

Name tests: `methodName_shouldExpectedBehavior_whenCondition`

**Do not add `@DisplayName`.** The method name itself, following the
`methodName_shouldExpectedBehavior_whenCondition` convention, is the
explanation — a separate display string is redundant and drifts out of sync
with the method name over time. Existing `@DisplayName` in the file you are
editing is legacy: remove it rather than matching it.

`// Given` / `// When` / `// Then` comments are **optional**. Use them when a test's setup is
  long enough that the phases aren't obvious; leave them out when blank lines already separate
  arrange, act and assert. Never write an empty `// Given` just to have the set. Put the effort into a name that's
descriptive on its own instead of adding an annotation on top of it.

```java
// ❌ Don't
@Test
@DisplayName("should return the expected result when the input is valid")
void methodName_shouldReturnExpectedResult_whenValidInput() { ... }

// ✅ Do
@Test
void methodName_shouldReturnExpectedResult_whenValidInput() { ... }
```

**Always use `var` for non-reassignable test variables:**

```java
@Test
void methodName_shouldReturnExpectedResult_whenValidInput() {
    // Given
    var input = createTestInput();
    var expectedValue = "expected";
    when(dependency.method(any())).thenReturn(expectedValue);

    // When
    var result = classUnderTest.methodName(input);

    // Then
    assertThat(result)
        .isNotNull()
        .satisfies(r -> {
            assertThat(r.getField()).isEqualTo(expectedValue);
        });

    verify(dependency, times(1)).method(any());
}
```

**Key principle:** Use `var` for all local test variables that are not reassigned. The type is always clear from the right-hand side in tests.

### 3.1. Parameterized Tests

For tests with identical structure but different input values, use `@ParameterizedTest` with value sources to reduce duplication:

**Benefits:**
- Reduces code repetition significantly
- Easier to maintain and extend
- Better test reporting (each parameter set runs as separate test)
- Single source of truth for test data

**Use `@ValueSource` for simple values:**
```java
@ParameterizedTest
@ValueSource(strings = {"value1", "value2", "value3"})
void methodName_shouldReturnTrue_whenInputMatches(String input) {
    // When
    var result = classUnderTest.methodName(input);

    // Then
    assertThat(result).isTrue();
}
```

**Use `@CsvSource` for multiple parameters:**
```java
@ParameterizedTest
@CsvSource({
    "10, true",
    "20, true",
    "5, false"
})
void methodName_shouldValidateRange_whenInputProvided(int value, boolean expected) {
    // When
    boolean result = classUnderTest.isValidRange(value);

    // Then
    assertThat(result).isEqualTo(expected);
}
```

**Use `@MethodSource` for complex objects:**
```java
@ParameterizedTest
@MethodSource("provideTestCases")
void methodName_shouldProcessInput_whenCalled(TestInput input, TestOutput expected) {
    // When
    var result = classUnderTest.process(input);

    // Then
    assertThat(result).isEqualTo(expected);
}

private static Stream<Arguments> provideTestCases() {
    return Stream.of(
        Arguments.of(new TestInput("test1"), new TestOutput("result1")),
        Arguments.of(new TestInput("test2"), new TestOutput("result2"))
    );
}
```

**When to use Parameterized Tests:**
- ✅ Testing same method with multiple valid values (enum constants, valid inputs)
- ✅ Testing boolean logic with true/false cases
- ✅ Testing edge cases with different boundaries
- ✅ Testing error scenarios with different invalid inputs

**When NOT to use Parameterized Tests:**
- ❌ Complex setup required for each test case
- ❌ Different assertions needed for different inputs
- ❌ Testing different methods or functionality

### 4. Test Coverage Requirements

For each public method, generate tests for:

**Happy Path:**
- Successful execution with valid inputs
- Correct return value/behavior
- Proper interaction with dependencies

**Edge Cases:**
- Null inputs (if not validated)
- Empty collections
- Boundary values
- Multiple invocations

**Error Cases:**
- Exception scenarios
- Validation failures
- Dependency failures

**Verification:**
- Mock interactions (`verify`)
- Argument values passed to dependencies (`ArgumentCaptor`)
- Order of operations (`inOrder`)
- State changes

### 5. AssertJ Patterns

Use fluent AssertJ assertions:

```java
// Basic
assertThat(result).isNotNull();
assertThat(result).isEqualTo(expected);
assertThat(result).isSameAs(sameObject);

// Fluent chaining
assertThat(result)
    .isNotNull()
    .satisfies(r -> {
        assertThat(r.getField1()).isEqualTo(value1);
        assertThat(r.getField2()).isNotEmpty();
    });

// Exceptions
assertThatThrownBy(() -> service.method())
    .isInstanceOf(ExpectedException.class)
    .hasMessage("Expected message")
    .isSameAs(thrownException);

// Collections
assertThat(list)
    .isNotEmpty()
    .hasSize(3)
    .extracting(Item::getName)
    .containsExactly("name1", "name2", "name3");

// Optional
assertThat(optional)
    .isPresent()
    .get()
    .isEqualTo(expectedValue);
```

### 6. Mockito Patterns

**Stubbing:**
```java
when(mock.method(any())).thenReturn(value);
when(mock.method(eq(specific))).thenReturn(specificValue);
when(mock.method(any(), any())).thenReturn(value);
```

**Verification:**
```java
verify(mock, times(1)).method(any());
verify(mock, never()).method(any());
verify(mock, atLeastOnce()).method(any());
```

**Argument Capture:**
```java
@Captor
private ArgumentCaptor<ArgumentType> captor;

// In test
verify(mock).method(captor.capture());
assertThat(captor.getValue()).isEqualTo(expected);
```

**Exception Throwing:**
```java
when(mock.method(any())).thenThrow(new RuntimeException("Error"));

// Or with doThrow for void methods
doThrow(new RuntimeException("Error"))
    .when(mock)
    .voidMethod(any());
```

**Order Verification:**
```java
var inOrder = org.mockito.Mockito.inOrder(mockOne, mockTwo);
inOrder.verify(mockOne).firstMethod();
inOrder.verify(mockTwo).secondMethod();
```

### 7. Test Data Setup

Create helper methods for test data. **Always use `var` for local variables** when creating test objects:

```java
private RequestHeaderData createRequestHeaderData() {
    var headerData = new RequestHeaderData();
    headerData.setCorrelationId("test-correlation-id");
    headerData.setRequestId("test-request-id");
    return headerData;
}

private PartyData createPartyData() {
    var partyData = new PartyData();
    partyData.setWorkflowId(WORKFLOW_ID);
    partyData.setCreateAccountReqType(createAccountReqType);
    return partyData;
}

// Use var for builder patterns in tests
private ContainerDto createTestContainer() {
    var container = GateInGateOutContainerDto.builder()
        .containerNumber("TEST123")
        .containerType("40HQ")
        .weight("15640")
        .releaseNumber("31728925")
        .build();
    return container;
}
```

**Important: Use `var` for all non-reassignable local variables** in test data setup and test methods:
- When creating test objects
- When assigning method results
- When using builders
- Anywhere the type is clear from the right-hand side

Or use test fixtures if they exist in the project.

### 8. Constants

Define test constants at class level:

```java
private static final String TEST_WORKFLOW_ID = "workflow-123";
private static final String TEST_APPLICATION_ID = "app-456";
private static final Long TEST_PARTY_ID = 789L;
```

## Output Format

1. **File location**: Place test in `src/test/java/` mirroring the source package structure
2. **File name**: `[ClassName]Test.java`
3. **Test count**: Aim for 3-7 tests per method depending on complexity
4. **Coverage**: Target all public methods, focusing on business logic

## Example Generation Request

```
User: "Generate tests for DigitalApplicationEventService"

Steps:
1. Read DigitalApplicationEventService.java
2. Identify dependencies and methods
3. Create DigitalApplicationEventServiceTest.java
4. Generate tests for each public method:
   - buildDigitalApplicationEventBy
5. Include happy path, edge cases, error scenarios
6. Verify mock interactions
7. Use AssertJ assertions
```

## Parameterized Tests Example: Before vs After

### ❌ BEFORE: Multiple duplicate tests
```java
@Test
void fromName_shouldReturnCaseCreation_whenNameMatches() {
    var result = EnumClass.fromName("CASE_CREATION");
    assertThat(result).isPresent()
            .get()
            .isEqualTo(EnumClass.CASE_CREATION);
}

@Test
void fromName_shouldReturnTechnicalError_whenNameMatches() {
    var result = EnumClass.fromName("TECHNICAL_ERROR");
    assertThat(result).isPresent()
            .get()
            .isEqualTo(EnumClass.TECHNICAL_ERROR);
}

@Test
void fromName_shouldReturnPendingEV_whenNameMatches() {
    var result = EnumClass.fromName("PENDING_EV");
    assertThat(result).isPresent()
            .get()
            .isEqualTo(EnumClass.PENDING_EV);
}

// ... 20 more similar tests ...
```

### ✅ AFTER: Single parameterized test
```java
@ParameterizedTest
@ValueSource(strings = {
    "CASE_CREATION",
    "TECHNICAL_ERROR",
    "PENDING_EV",
    "CASE_CLOSURE_ACCEPT",
    // ... all 23 values ...
})
void fromName_shouldReturnCorrectEnum_whenNameMatches(String enumName) {
    // When
    var result = EnumClass.fromName(enumName);

    // Then
    assertThat(result).isPresent()
            .get()
            .isEqualTo(EnumClass.valueOf(enumName));
}
```

**Results:**
- Reduced from 23 tests to 1 parameterized test
- Each value still runs as a separate test case
- Much easier to add new values (just add to array)
- Cleaner, more maintainable code
- Better test naming clarity

## Quality Checklist

Before presenting tests, verify:
- [ ] Uses `@ExtendWith(MockitoExtension.class)` for mocked tests
- [ ] All dependencies are `@Mock`ed
- [ ] Class under test initialized in `@BeforeEach`
- [ ] Test names follow `method_should_when` pattern
- [ ] **No `@DisplayName` annotations** — the method name is the explanation
- [ ] Arrange / act / assert separated (Given/When/Then comments optional)
- [ ] **All non-reassignable variables use `var`** (essential)
- [ ] AssertJ assertions (`assertThat`)
- [ ] Mock interactions verified
- [ ] Exception cases covered
- [ ] Argument captors used where needed
- [ ] No hardcoded values (use constants)
- [ ] Helper methods for complex test data
- [ ] **Parameterized tests** used for repetitive test structures
- [ ] **Appropriate source** chosen (`@ValueSource`, `@CsvSource`, `@MethodSource`)
- [ ] **No duplicate logic** across similar test cases

## Notes

- Check for existing test fixtures in `src/test/.../fixture/` directory
- Reuse fixture classes when available
- Match the testing style of existing tests in the project
- Focus on behavior verification, not implementation details
- Keep tests independent and deterministic