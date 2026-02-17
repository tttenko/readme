```java
@Mapper(componentModel = "spring")
public interface FxRateEntityMapper {

    @Mapping(target = "id", ignore = true) // если id генерится БД/hibernate
    @Mapping(target = "requestUid", source = "requestUid")
    @Mapping(target = "requestTime", source = "requestTime")

    @Mapping(target = "fxRateSubType", source = "fxRateXmlDto.fxRateSubType")

    @Mapping(target = "fromCurrencyCode", source = "fxRateXmlDto.code1")
    @Mapping(target = "fromCurrencyIsoNum", source = "fxRateXmlDto.isoNum1")

    @Mapping(target = "toCurrencyCode", source = "fxRateXmlDto.code2")
    @Mapping(target = "toCurrencyIsoNum", source = "fxRateXmlDto.isoNum2")

    @Mapping(target = "useDate", source = "fxRateXmlDto.useDate")
    @Mapping(target = "lotSize", source = "fxRateXmlDto.lotSize")
    @Mapping(target = "value", source = "fxRateXmlDto.value")
    FxRateEntity toEntity(String requestUid, ZonedDateTime requestTime, FxRateXmlDto fxRateXmlDto);
}
```
