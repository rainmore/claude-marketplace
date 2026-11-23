---
name: java-code-placement
description: Where Java logic and constants belong — no new utility classes (*Util, *Utils, *Helper, @UtilityClass, classes of only static methods) and no new public constants (public static final fields, interface constants). Gives the owning type, injectable bean, enum, record or properties alternative for each case. Use whenever you are about to add a static method, a helper class, a shared function, or a constant, or when reviewing code that does.
---

# Code Placement: no utility classes, no public constants

Both are symptoms of the same thing: behaviour or data that hasn't found the type it belongs to.
A `*Util` class grows into an unrelated grab bag, can't be mocked, and hides dependencies such as
the clock or the time zone. A public constant spreads one class's internal detail across the
codebase, so changing it means changing every caller.

These rules apply to **new code**. Existing utility classes and public constants stay as they are
unless the user asks to change them. Don't add to them, and don't copy their style.

## 1. Utility classes

**Never create** a class named `*Util`, `*Utils`, `*Helper` or `*Helpers`, a class annotated
`@UtilityClass`, or a class whose methods are all `static`. Don't add new static methods to an
existing one either.

Before writing a static helper, ask: **"Which type owns this?"** Then place it with the first rule
that fits:

| The logic... | Put it | Instead of |
| --- | --- | --- |
| is about one value's data | a method on that type (record, entity, enum) | `DateUtil.contains(range, date)` → `range.contains(date)` |
| parses or validates a domain string | a record with a `parse`/`of` factory and checks in its compact constructor | `ContainerNumberValidator.isValid(s)` → `ContainerNumber.parse(s)` |
| needs the clock, configuration, the zone or another bean | an injectable `@Component` | `DateUtil.now()` → inject `Clock`; `DateUtil.format(...)` → a `DisplayDateFormatter` bean built from properties |
| is used by only one class | a `private` method in that class (`private static` is fine) | a public helper "in case it's reused" |
| extends the framework (`Converter`, `Formatter`, `Validator`) | its own top-level class, registered as a bean or in a `WebMvcConfigurer` | nested `public static class` inside a util |
| already exists in the JDK or a dependency | call it directly (`Objects`, `String.isBlank`, `List.copyOf`, `Duration`, `java.nio.file.Files`, commons-lang3 `StringUtils`) | a wrapper that renames it |
| belongs to many types of one kind | a small interface with a `default` method, or a base class the types already share | a static method taking the type as its first parameter |

### Static methods that are allowed

- `private static` helpers inside the class that uses them.
- Static factories **on their own type**: `ReportDateRange.ofMonth(month)`, `ContainerNumber.parse(s)`,
  `RobotHealthDto.of(health, now)`.
- `main`, and the compact constructor and factories of records.

### Review signal

A static method whose first parameter is the type it operates on (`format(LocalDateTime x)`,
`isEmpty(List<?> list)`) is an instance method in the wrong place, or a wrapper around a library
call.

## 2. Public constants

**Every constant starts as `private static final`.** Widen it only through the following table, and
never to `public` just because another class "might need it".

| The value is... | Make it | Not |
| --- | --- | --- |
| used only in this class (including in its annotations, e.g. `@GetMapping(URI_LIST)`) | `private static final` | `public` |
| needed by a same-package test or collaborator (native SQL in `*Sql`, message templates) | package-private `static final` (no modifier); keep the test in the same package | `public` |
| one of a closed set (statuses, kinds, authorities, report columns, channels) | an `enum`, with any code or label as a field | `public static final String STATUS_PENDING` |
| a setting someone might change (timeouts, sizes, formats, recipients, URLs of other systems) | a component of a `@ConfigurationProperties` record | a constant |
| a value another class needs in order to *do something* | a method on the owner that does that thing (`View.redirect(uri, params)`, `invoice.isOverdue(clock)`) | exposing the value for the caller to use |
| a URI a template needs | a record of URIs the controller adds to the model (section 3) | `T(Controller).URI_X` in the template |
| a URI a test needs | the literal path in the test, since the URL is the public contract under test | `Controller.URI_X` |
| shared default data (a default slug, a system account name) | a method or field on the owning service or enum | a public string |

Also:
- **No constants in interfaces.** Interface fields are implicitly `public static final`; use an enum.
- **Never `public static final` with a mutable or environment-dependent value**: `List`, `Map`,
  arrays, `ZoneId.systemDefault()`, `new SimpleDateFormat(...)`. They are shared mutable state or
  are evaluated once, at class-load time.
- **No public singleton fields** (`Mapper.INSTANCE`). Inject the bean; in tests, construct it or
  call the framework factory (`Mappers.getMapper(XMapper.class)`).

### The one unavoidable case

An annotation attribute in **another** class that needs a compile-time constant from this one (for
example a Spring Integration channel name in `@ServiceActivator(inputChannel = ...)`). First try to
move the two classes into one package and make the constant package-private. If they really live
in different packages, a `public static final` on the class that **defines** the thing (the config
class that declares the channel bean) is acceptable. Add a comment naming the annotation that
needs it.

## 3. URIs for templates without public constants

```java
@Controller
@RequiredArgsConstructor
public class InvoiceViewController extends BaseViewController {

    private static final String URI_LIST = "/invoices";
    private static final String URI_ADD = "/invoices/add";
    private static final String URI_EDIT = "/invoices/{id}";
    private static final String VIEW_LIST = "bulma/invoice/list";

    /** The URIs this controller's templates link to. */
    public record Uris(String list, String add, String edit) {
    }

    @ModelAttribute("uris")
    Uris uris() {
        return new Uris(URI_LIST, URI_ADD, URI_EDIT);
    }

    @GetMapping(URI_LIST)
    public String list(...) { ... }
}
```

```html
<!--/*@thymesVar id="uris" type="com.example.app.invoice.controller.InvoiceViewController.Uris"*/-->
<a th:href="@{${uris.add}}">Add</a>
<a th:href="@{${uris.edit}(id=${item.id})}">Edit</a>
```

The constants stay private. The template gets a typed, documented record, and `@thymesVar` keeps IDE
checking. Tests assert against literal paths (`get("/invoices")`).

## 4. Enums instead of constant groups

```java
public enum SettingAuthority implements GrantedAuthority {
    PRICING_ADMIN("settings.pricing.admin"),
    PRICING_READ("settings.pricing.read");

    private final String authority;

    SettingAuthority(String authority) {
        this.authority = authority;
    }

    @Override
    public String getAuthority() {
        return authority;
    }
}
```

Enum constants are public, and that's fine: an enum is a type with a closed set of values, which the
compiler checks. Callers can't pass an arbitrary string, and a `switch` over it is exhaustive.

If the project's permission check takes its own authority type rather than `GrantedAuthority`, keep
that type as a private field of the enum and expose it with a method (`PricingAuthority.ADMIN.authority()`).
New authorities always go in an enum, never in interface constants.

## Checklist

- [ ] No new `*Util`/`*Utils`/`*Helper`, `@UtilityClass`, or all-static class
- [ ] Each new static method is `private`, or a factory on its own type
- [ ] Logic that needs the clock, config or beans lives in an injectable component
- [ ] Each new constant is `private` or package-private; closed sets are enums; settings are properties
- [ ] No interface constants, no mutable or environment-dependent `static final` values, no `INSTANCE` fields
- [ ] Templates get URIs from a model record; tests use literal paths
