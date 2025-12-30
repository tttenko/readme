```java
@Override
    @Mapping(target = "uuid",        expression = "java(getValueOrNull(values, ITEM_ID))")
    @Mapping(target = "alpha2Code",  expression = "java(getValueText(attributes, SLUG_ALPHA2))")
    @Mapping(target = "shortName",   expression = "java(getValueText(attributes, SLUG_SHORT_NAME))")
    @Mapping(target = "fullName",    expression = "java(getValueText(attributes, SLUG_FULL_NAME))")
    @Mapping(target = "codeCountry", expression = "java(getValueText(attributes, SLUG_CODE_COUNTRY))")
    @Mapping(target = "alpha3Code",  expression = "java(getValueText(attributes, SLUG_ALPHA3))")
    CountryDto mapValuesToDto(Map<String, Object> values,
                              Map<String, Map<String, Object>> attributes);
```
