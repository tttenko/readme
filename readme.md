```java
@Mapper(config = MdMapperConfig.class)
public interface CurrencyMapperMs extends DataMapperWithAttribute<CurrencyDto> {

    String NAME = "name";
    String ALPHABETIC_ISO_CODE = "alphabeticIsoCode";
    String DIGITAL_ISO_CODE = "digitalIsoCode";
    String CODE_AST = "codeAst";
    String ID = "id";

    @Override
    @Mapping(target = "id",            expression = "java(getValueOrNull(values, ID))")
    @Mapping(target = "currencyName",  expression = "java(getValueText(attributes, NAME))")
    @Mapping(target = "currencyCode",  expression = "java(getValueText(attributes, ALPHABETIC_ISO_CODE))")
    @Mapping(target = "codeAst",       expression = "java(getValueText(attributes, CODE_AST))")
    @Mapping(target = "isoCode",       expression = "java(getValueText(attributes, DIGITAL_ISO_CODE))")
    // эти два поля заполним после маппинга
    @Mapping(target = "currencySymbol", ignore = true)
    @Mapping(target = "currencySymbolCode", ignore = true)
    CurrencyDto mapValuesToDto(Map<String, Object> values,
                               Map<String, Map<String, Object>> attributes);

    @AfterMapping
    default void enrichSymbols(@MappingTarget CurrencyDto dto) {
        String code = dto.getCurrencyCode();
        dto.setCurrencySymbol(CurrencySymbolDictionary.getSymbolByCode(code).orElse(null));
        dto.setCurrencySymbolCode(CurrencySymbolDictionary.getSymbolCodeByCode(code).orElse(null));
    }
}
```
