---
name: mapstruct-generator
description: Generate MapStruct mappers between DTOs with proper annotations, named qualifiers, custom mappings, and null handling. Use when asked to create a mapper, convert between types, or transform DTOs.
disable-model-invocation: false
---

# MapStruct Mapper Generator

Generate MapStruct mappers following enterprise patterns.

## When to Use

- User asks to "create a mapper", "generate MapStruct mapper"
- User requests "map SourceDto to TargetDto"
- User wants to "convert between types", "transform objects"

## Mapper Generation Process

### 1. Analyze Source and Target Types

Before generating:
- Read both source and target class files
- Identify matching fields (by name and type)
- Identify fields needing custom mapping
- Identify fields needing transformation
- Check for nested objects
- Identify collection mappings

### 2. Basic Mapper Template

```java
package com.macquarie.bfs.integration.trust.originations.workflow.mapper.[domain];

import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.MappingConstants.ComponentModel;
import org.mapstruct.Named;

@Mapper(componentModel = ComponentModel.SPRING)
public interface [Name]Mapper {
    @Mapping(target = "targetField", source = "sourceField")
    TargetType toTargetType(SourceType source);
}
```

**Essential Elements:**

1. **`@Mapper(componentModel = ComponentModel.SPRING)`**
   - Makes mapper a Spring bean
   - Enables dependency injection

2. **No `INSTANCE` field**
   - Inject the mapper as a Spring bean. A public `INSTANCE` singleton is a public constant
     (see `java-code-placement`).
   - In unit tests, create it with `Mappers.getMapper(MapperName.class)`.

3. **Method Naming**
   - Pattern: `to[TargetType]`, `from[SourceType]`
   - Examples: `toDto`, `toEntity`, `toCmaRequest`, `fromDomain`

### 3. Field Mapping Patterns

#### Pattern 1: Direct Mapping (Same Names)

Fields with same name map automatically (no annotation needed):

```java
// Source: { name, email, age }
// Target: { name, email, age }

// Automatic mapping - no @Mapping needed
TargetDto toDto(SourceDto source);
```

#### Pattern 2: Different Field Names

```java
@Mapping(target = "targetFieldName", source = "sourceFieldName")
@Mapping(target = "userId", source = "id")
@Mapping(target = "fullName", source = "name")
TargetDto toDto(SourceDto source);
```

#### Pattern 3: Nested Object Mapping

```java
@Mapping(target = "userName", source = "user.name")
@Mapping(target = "userEmail", source = "user.email")
@Mapping(target = "addressCity", source = "address.city")
TargetDto toDto(SourceDto source);
```

#### Pattern 4: Constant Values

```java
@Mapping(target = "type", constant = "USER")
@Mapping(target = "status", constant = "ACTIVE")
@Mapping(target = "version", constant = "1.0")
TargetDto toDto(SourceDto source);
```

#### Pattern 5: Ignore Fields

```java
@Mapping(target = "internalId", ignore = true)
@Mapping(target = "createdDate", ignore = true)
TargetDto toDto(SourceDto source);
```

### 4. Custom Mapping Methods

For complex transformations, use `@Named` methods:

```java
@Mapper(componentModel = ComponentModel.SPRING)
public interface UserMapper {

    @Mapping(target = "accountType", source = "trustDetails", qualifiedByName = "mapAccountType")
    @Mapping(target = "fullAddress", source = "address", qualifiedByName = "formatAddress")
    TargetDto toDto(SourceDto source);

    @Named("mapAccountType")
    default AccountType mapAccountType(TrustDetails trustDetails) {
        if (trustDetails == null) {
            return null;
        }
        return TrustTypeEnum.SMSF.equals(trustDetails.getTrustType())
                ? AccountType.SUPERANNUATION
                : AccountType.STANDARD;
    }

    @Named("formatAddress")
    default String formatAddress(Address address) {
        if (address == null) {
            return null;
        }
        return String.format("%s, %s, %s %s",
                address.getStreet(),
                address.getCity(),
                address.getState(),
                address.getPostalCode());
    }
}
```

**Custom Method Conventions:**

1. **Use `default` methods** in interface
2. **Prefix with qualifier name**: `map*`, `convert*`, `format*`, `transform*`
3. **Add `@Named` annotation** with qualifier string
4. **Reference with `qualifiedByName`** in `@Mapping`
5. **Always handle null inputs** explicitly

### 5. Enum Mapping

#### Simple Enum Mapping (Same Values)

```java
// Automatic if enum values match
TargetEnum toTargetEnum(SourceEnum source);
```

#### Custom Enum Mapping

```java
@Named("mapTaxPreference")
default String mapTaxPreference(TaxApplyPreferenceEnum taxPreference) {
    if (taxPreference == null) {
        return null;
    }
    return switch (taxPreference) {
        case TFN -> "TFN_SUPPLIED";
        case ABN -> "ABN_SUPPLIED";
        case TAX_EXEMPTION -> "TFN_EXEMPTED";
        case NOT_PROVIDED -> "TFN_NOT_SUPPLIED";
        default -> null;
    };
}
```

Or using `@ValueMapping`:

```java
import org.mapstruct.ValueMapping;

@ValueMapping(source = "TFN", target = "TFN_SUPPLIED")
@ValueMapping(source = "ABN", target = "ABN_SUPPLIED")
@ValueMapping(source = "TAX_EXEMPTION", target = "TFN_EXEMPTED")
@ValueMapping(source = "NOT_PROVIDED", target = "TFN_NOT_SUPPLIED")
TargetTaxEnum mapTaxEnum(SourceTaxEnum source);
```

### 6. Collection Mapping

```java
// List mapping
List<TargetDto> toDtoList(List<SourceDto> sources);

// Set mapping
Set<TargetDto> toDtoSet(Set<SourceDto> sources);

// Map mapping
@MapMapping(valueDateFormat = "dd.MM.yyyy")
Map<String, TargetDto> toDtoMap(Map<String, SourceDto> sources);
```

### 7. Multiple Source Parameters

```java
@Mapping(target = "field1", source = "source1.field")
@Mapping(target = "field2", source = "source2.field")
@Mapping(target = "field3", source = "source3.field")
TargetDto toDto(Source1 source1, Source2 source2, Source3 source3);
```

### 8. Using Other Mappers

When mapping nested objects that have their own mappers:

```java
@Mapper(componentModel = ComponentModel.SPRING, uses = {AddressMapper.class, PhoneMapper.class})
public interface UserMapper {
    // AddressMapper and PhoneMapper will be used automatically for nested objects
    @Mapping(target = "homeAddress", source = "address")
    @Mapping(target = "primaryPhone", source = "phone")
    UserDto toDto(User user);
}
```

### 9. Conditional Mapping

Using expressions:

```java
@Mapping(target = "status", expression = "java(source.isActive() ? \"ACTIVE\" : \"INACTIVE\")")
@Mapping(target = "displayName", expression = "java(formatDisplayName(source))")
UserDto toDto(User source);

default String formatDisplayName(User user) {
    return user.getFirstName() + " " + user.getLastName();
}
```

### 10. Null Handling Strategies

```java
@Mapper(
    componentModel = ComponentModel.SPRING,
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface UpdateMapper {
    // Null source properties won't overwrite target
    void updateEntity(@MappingTarget EntityType target, DtoType source);
}
```

### 11. Update Methods

For updating existing objects:

```java
import org.mapstruct.MappingTarget;

@Mapping(target = "id", ignore = true)  // Don't update ID
@Mapping(target = "createdDate", ignore = true)  // Don't update creation date
void updateEntityFromDto(@MappingTarget Entity target, Dto source);
```

### 12. Complex Example

```java
package com.macquarie.bfs.integration.trust.originations.workflow.mapper.cma;

import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.MappingConstants.ComponentModel;
import org.mapstruct.Named;

import com.macquarie.bfs.integration.trust.originations.workflow.dto.digital.CreateAccountReqType;
import com.macquarie.bfs.integration.trust.originations.workflow.dto.digital.CreateAccountReqTypeTrustDetails;
import com.macquarie.bfs.integration.trust.originations.workflow.dto.digital.TaxDetailsType;
import com.macquarie.bfs.integration.trust.originations.workflow.generated.model.cma.CMAAccountSubType;
import com.macquarie.bfs.integration.trust.originations.workflow.generated.model.cma.CMAAccountToCreate;
import com.macquarie.bfs.integration.trust.originations.workflow.generated.model.cma.CMACreateCashAccountRequest;

@Mapper(componentModel = ComponentModel.SPRING)
public interface CmaOriginationMapper {

    @Mapping(target = "applicationId", source = "applicationId")
    @Mapping(target = "linkedAccountProductCodes", source = "requestedAccountDetails.bundledProductIds")
    @Mapping(target = "accountType", constant = "TRUST")
    @Mapping(target = "accountSubType", source = "trustDetails", qualifiedByName = "mapAccountSubType")
    @Mapping(target = "accountDetail", source = "domainCreateAccountReqType", qualifiedByName = "toCmaAccountToCreate")
    CMACreateCashAccountRequest toCmaRequest(CreateAccountReqType domainCreateAccountReqType);

    @Named("mapAccountSubType")
    default CMAAccountSubType mapAccountSubType(CreateAccountReqTypeTrustDetails trustDetails) {
        if (trustDetails == null) {
            return null;
        }
        return CreateAccountReqTypeTrustDetails.TrustTypeEnum.ATO_REGULATED_SELF_MANAGED_SUPERANNUATION_FUND
                .equals(trustDetails.getTrustType())
                ? CMAAccountSubType.SMSF
                : null;
    }

    @Named("toCmaAccountToCreate")
    @Mapping(target = "controllingPartiesBeneficiaries", source = "trustDetails.areAllControllingPartiesBeneficiaries")
    @Mapping(target = "controllingPartiesRelated", source = "trustDetails.areAllControllingPartiesRelated")
    @Mapping(target = "sourceOfFunds", source = "trustDetails.amlDetails.sourceOfFunds")
    @Mapping(target = "purposeOfInvestment", source = "trustDetails.amlDetails.purposeOfInvestment")
    @Mapping(target = "productCode", source = "requestedAccountDetails.productId")
    CMAAccountToCreate toCmaAccountToCreate(CreateAccountReqType domainCreateAccountReqType);

    @Named("mapTfnOption")
    default String mapTfnOption(TaxDetailsType.TaxApplyPreferenceEnum taxApplyPreference) {
        if (taxApplyPreference == null) {
            return "TFN_NOT_SUPPLIED";
        }
        return switch (taxApplyPreference) {
            case TFN -> "TFN_SUPPLIED";
            case TAX_EXEMPTION -> "TFN_EXEMPTED";
            default -> "TFN_NOT_SUPPLIED";
        };
    }

    @Named("mapApplyAbn")
    default Boolean mapApplyAbn(TaxDetailsType.TaxApplyPreferenceEnum taxApplyPreference) {
        return TaxDetailsType.TaxApplyPreferenceEnum.ABN.equals(taxApplyPreference);
    }
}
```

### 13. Testing MapStruct Mappers

Generate tests for mappers:

```java
@ExtendWith(MockitoExtension.class)
class CmaOriginationMapperTest {

    private CmaOriginationMapper mapper;

    @BeforeEach
    void setUp() {
        mapper = Mappers.getMapper(CmaOriginationMapper.class);
    }

    @Test
    void toCmaRequest_shouldMapAllFields_whenValidInput() {
        // Given
        var source = createSourceObject();

        // When
        var result = mapper.toCmaRequest(source);

        // Then
        assertThat(result)
                .isNotNull()
                .satisfies(r -> {
                    assertThat(r.getApplicationId()).isEqualTo(source.getApplicationId());
                    assertThat(r.getAccountType()).isEqualTo("TRUST");
                    assertThat(r.getAccountSubType()).isNotNull();
                });
    }

    @Test
    void mapAccountSubType_shouldReturnSMSF_whenTrustTypeIsSMSF() {
        // Given
        var trustDetails = new CreateAccountReqTypeTrustDetails();
        trustDetails.setTrustType(TrustTypeEnum.ATO_REGULATED_SELF_MANAGED_SUPERANNUATION_FUND);

        // When
        var result = mapper.mapAccountSubType(trustDetails);

        // Then
        assertThat(result).isEqualTo(CMAAccountSubType.SMSF);
    }

    @Test
    void mapAccountSubType_shouldReturnNull_whenTrustDetailsIsNull() {
        // When
        var result = mapper.mapAccountSubType(null);

        // Then
        assertThat(result).isNull();
    }

    @Test
    void mapAccountSubType_shouldReturnNull_whenTrustTypeIsNotSMSF() {
        // Given
        var trustDetails = new CreateAccountReqTypeTrustDetails();
        trustDetails.setTrustType(TrustTypeEnum.OTHER);

        // When
        var result = mapper.mapAccountSubType(trustDetails);

        // Then
        assertThat(result).isNull();
    }
}
```

### 14. Package Structure

Place mappers in appropriate package:

```
com.macquarie.bfs.integration.trust.originations.workflow
├── mapper/
│   ├── cma/
│   │   └── CmaOriginationMapper.java
│   ├── mdm/
│   │   └── MdmPartyMapper.java
│   ├── digital/
│   │   └── DigitalApplicationMapper.java
│   └── pega/
│       └── PegaEntityMapper.java
```

Group mappers by target system or domain.

## Generation Checklist

Before generating:
- [ ] Source type identified and read
- [ ] Target type identified and read
- [ ] Field mappings determined
- [ ] Custom transformations identified
- [ ] Null handling strategy determined

After generating:
- [ ] `@Mapper(componentModel = ComponentModel.SPRING)` present
- [ ] No `INSTANCE` field; tests use `Mappers.getMapper(...)`
- [ ] All fields mapped (explicitly or implicitly)
- [ ] Custom methods have `@Named` annotation
- [ ] Custom methods handle null inputs
- [ ] Appropriate `qualifiedByName` references
- [ ] Method names follow `to*`/`from*` convention
- [ ] Tests generated for custom methods
- [ ] Tests cover null cases
- [ ] Tests cover all enum mappings

## Example Generation

```
User: "Create a mapper from CreateAccountReqType to CMACreateCashAccountRequest"

Steps:
1. Read CreateAccountReqType.java
2. Read CMACreateCashAccountRequest.java
3. Identify matching fields
4. Identify fields needing transformation
5. Create CmaOriginationMapper.java
6. Add @Mapping annotations
7. Create custom @Named methods for complex mappings
8. Add null handling
9. Generate tests: CmaOriginationMapperTest.java
10. Test all custom mapping methods
```

## Common Pitfalls to Avoid

- Forgetting `componentModel = ComponentModel.SPRING`
- Adding a public `INSTANCE` field
- Not handling null in custom methods
- Using `source` when mapping entire object (use `source = "."` or method parameter)
- Circular references in nested mappings
- Missing `@Named` annotation on custom methods
- Not testing custom mapping methods
- Ignoring fields without explicit `ignore = true`

## Notes

- MapStruct generates implementation at compile time
- Check generated code in `target/generated-sources/annotations`
- Use `uses` parameter to compose mappers
- Consider `@BeforeMapping` and `@AfterMapping` for complex logic
- For bi-directional mapping, create separate methods: `toDto()` and `toEntity()`
- Test edge cases: null objects, empty collections, null nested objects