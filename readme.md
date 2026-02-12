```java
@Configuration
public class XmlConfig {

  @Bean
  public XmlMapper xmlMapper() {
    XmlMapper mapper = new XmlMapper();
    mapper.registerModule(new com.fasterxml.jackson.datatype.jsr310.JavaTimeModule());
    mapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
    mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    return mapper;
  }
}

@JacksonXmlRootElement(localName = "FXRates")
@JsonIgnoreProperties(ignoreUnknown = true)
public class FxRateXmlDto {

  @JacksonXmlProperty(localName = "IsPublic")
  public String isPublic;

  @JacksonXmlProperty(isAttribute = true, localName = "FXRateSubType")
  public String fxRateSubType;

  @JacksonXmlProperty(isAttribute = true, localName = "Code1")
  public String code1;

  @JacksonXmlProperty(isAttribute = true, localName = "ISONum1")
  public String isoNum1;

  @JacksonXmlProperty(isAttribute = true, localName = "Code2")
  public String code2;

  @JacksonXmlProperty(isAttribute = true, localName = "ISONum2")
  public String isoNum2;

  @JacksonXmlProperty(isAttribute = true, localName = "PublicationDate")
  public LocalDateTime publicationDate;

  @JacksonXmlProperty(isAttribute = true, localName = "UseDate")
  public LocalDateTime useDate;

  @JacksonXmlProperty(isAttribute = true, localName = "LotSize")
  public BigDecimal lotSize;

  @JacksonXmlProperty(isAttribute = true, localName = "Value")
  public BigDecimal value;
}

@JacksonXmlRootElement(localName = "PutEODPriceNf")
@JsonIgnoreProperties(ignoreUnknown = true)
public class PutEodPriceNfDto {

  @JacksonXmlProperty(localName = "RqUID")
  public UUID rquid;

  @JacksonXmlProperty(localName = "RqTm")
  public LocalDateTime rqtm;

  @JacksonXmlElementWrapper(useWrapping = false)
  @JacksonXmlProperty(localName = "FXRates")
  public List<FxRateXmlDto> fxRates;
}

public final class FxRateEntityMapper {

  private FxRateEntityMapper() {}

  /**
   * Маппит 1 FXRates в entity + делает нормализации:
   * - trim
   * - upper
   * - RUR -> RUB (только для code2)
   *
   * Возвращает null, если запись "не годится" для сохранения (нет ключевых полей / lotSize=0).
   * Если хочешь вместо null — кидать исключение, скажи.
   */
  public static FxRateEntity toEntity(UUID rquid, LocalDateTime rqtm, FxRateXmlDto x) {
    if (x == null) return null;
    if (rquid == null || rqtm == null) {
      // это скорее ошибка envelope-а, но пусть mapper защищает
      return null;
    }

    String subType = upperTrimToNull(x.fxRateSubType);
    String code1   = upperTrimToNull(x.code1);
    String code2   = normalizeRub(upperTrimToNull(x.code2));

    LocalDateTime useDate = x.useDate; // если в DTO LocalDateTime
    BigDecimal lotSize    = x.lotSize; // если в DTO BigDecimal
    BigDecimal value      = x.value;

    // ключевые поля
    if (subType == null || code1 == null || code2 == null || useDate == null) return null;

    // защита от мусора: lotSize=0 (если по домену это недопустимо)
    if (lotSize != null && lotSize.compareTo(BigDecimal.ZERO) == 0) return null;

    FxRateEntity e = new FxRateEntity();
    e.setRquid(rquid);
    e.setRqtm(rqtm);

    e.setFxRateSubType(subType);
    e.setCode1(code1);
    e.setIsoNum1(trimToNull(x.isoNum1));

    e.setCode2(code2);
    e.setIsoNum2(trimToNull(x.isoNum2));

    e.setUseDate(useDate);
    e.setLotSize(lotSize);
    e.setValue(value);

    return e;
  }

  private static String upperTrimToNull(String s) {
    String t = trimToNull(s);
    return t == null ? null : t.toUpperCase(Locale.ROOT);
  }

  private static String trimToNull(String s) {
    if (s == null) return null;
    String t = s.trim();
    return t.isEmpty() ? null : t;
  }

  private static String normalizeRub(String code2) {
    if (code2 == null) return null;
    return "RUR".equals(code2) ? "RUB" : code2;
  }
}


```
