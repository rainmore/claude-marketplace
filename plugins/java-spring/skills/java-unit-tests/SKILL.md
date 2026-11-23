---
name: java-unit-tests
description: Write and review Java unit tests using JUnit 5, Mockito, and AssertJ. Use when creating tests, adding coverage, or reviewing test code. Test method names must be self-describing — @DisplayName is prohibited.
---

# Java Unit Tests

## Naming is the documentation — never use `@DisplayName`

**Do not annotate tests with `@DisplayName`.** The method name is the description. A
`@DisplayName` string sitting next to a vague method name means the name has failed at its job,
and the two drift apart the moment someone edits one and not the other.

```java
// Wrong — the annotation carries meaning the name should have
@Test
@DisplayName("should refuse approval and queue nothing when the gate-in EDI is missing")
void approve_shouldRefuse() { }

// Right — the name carries it, and survives into stack traces, IDE runners, CI
// output, --tests filters, and grep
@Test
void approve_shouldRefuseAndQueueNothing_whenGateInEdiIsBlank() { }
```

This applies equally to `@Nested` classes: name the class `ApproveTests`, not a vague class with
`@DisplayName("approve()")`.

If a name would be unbearably long, the test is asserting too much. Split it.

### Existing `@DisplayName` is legacy — remove it when you touch the file

Older tests may still carry `@DisplayName`. Do not copy that style from neighbouring tests: the
ban overrides existing code. When you edit a test class, delete every `@DisplayName` in it, and if
the annotation held meaning the method name lacks, move that meaning into the name first.

## Test body layout

Separate arrange, act and assert with blank lines.
- `// Given` / `// When` / `// Then` comments are **optional**. Use them when a test's setup is
  long enough that the phases aren't obvious; leave them out when blank lines already separate
  arrange, act and assert. Never write an empty `// Given` just to have the set.

### Name shape

`methodUnderTest_shouldExpectedOutcome_whenCondition`

- The `_when` clause is optional when there is only one path.
- State the **outcome**, not the mechanic: `shouldRollBackTheApproval`, not `shouldThrow`.
- Never number tests (`approve_test1`) or restate the class (`testApproveService`).