```java/**
@Mapper(componentModel = "spring")
public interface FxRateEntityMapper {

    ZoneId SOURCE_ZONE = ZoneId.of("Europe/Moscow"); // <-- выбери вашу "зону источника"

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "requestUid", source = "requestUid")
    @Mapping(target = "requestTime", source = "requestTime", qualifiedByName = "toZoned")

    @Mapping(target = "fxRateSubType", source = "fxRateXmlDto.fxRateSubType")

    @Mapping(target = "fromCurrencyCode", source = "fxRateXmlDto.code1")
    @Mapping(target = "fromCurrencyIsoNum", source = "fxRateXmlDto.isoNum1")

    @Mapping(target = "toCurrencyCode", source = "fxRateXmlDto.code2")
    @Mapping(target = "toCurrencyIsoNum", source = "fxRateXmlDto.isoNum2")

    @Mapping(target = "useDate", source = "fxRateXmlDto.useDate", qualifiedByName = "toZoned")
    @Mapping(target = "lotSize", source = "fxRateXmlDto.lotSize")
    @Mapping(target = "value", source = "fxRateXmlDto.value")
    FxRateEntity toEntity(String requestUid, LocalDateTime requestTime, FxRateXmlDto fxRateXmlDto);

    @Named("toZoned")
    default ZonedDateTime toZoned(LocalDateTime dt) {
        return dt == null ? null : dt.atZone(SOURCE_ZONE);
    }
}
```
