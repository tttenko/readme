```java

 public final class XmlConverters {

  private XmlConverters() {}

  // ======================
  // String normalizers
  // ======================

  public static class TrimToNull extends StdConverter<String, String> {
    @Override public String convert(String s) {
      return trimToNull(s);
    }
  }

  public static class UpperTrimToNull extends StdConverter<String, String> {
    @Override public String convert(String s) {
      String t = trimToNull(s);
      return t == null ? null : t.toUpperCase(Locale.ROOT);
    }
  }

  /** Upper + trim + RUR -> RUB */
  public static class NormalizeRubUpper extends StdConverter<String, String> {
    @Override public String convert(String s) {
      String t = trimToNull(s);
      if (t == null) return null;
      t = t.toUpperCase(Locale.ROOT);
      return "RUR".equals(t) ? "RUB" : t;
    }
  }

  // ======================
  // Simple type converters
  // ======================

  public static class ToUuid extends StdConverter<String, UUID> {
    @Override public UUID convert(String s) {
      String t = trimToNull(s);
      if (t == null) return null;
      return UUID.fromString(t);
    }
  }

  public static class ToBigDecimal extends StdConverter<String, BigDecimal> {
    @Override public BigDecimal convert(String s) {
      String t = trimToNull(s);
      if (t == null) return null;
      return new BigDecimal(t);
    }
  }

  // ======================
  // Date/Time converters
  // ======================

  /** "2026-02-09T18:00:06" или "2026-02-09T18:00:06Z" / "+03:00" -> LocalDateTime */
  public static class ToLocalDateTimeLenient extends StdConverter<String, LocalDateTime> {
    @Override public LocalDateTime convert(String s) {
      return toLocalDateTimeLenient(s);
    }
  }

  /** "2026-02-10" или "2026-02-10T00:00:00" или "2026-02-10T00:00:00Z"/"+03:00" -> LocalDate */
  public static class ToLocalDateLenient extends StdConverter<String, LocalDate> {
    @Override public LocalDate convert(String s) {
      return toLocalDateLenient(s);
    }
  }

  // ======================
  // private helpers
  // ======================

  private static String trimToNull(String s) {
    if (s == null) return null;
    String t = s.trim();
    return t.isEmpty() ? null : t;
  }

  private static LocalDateTime toLocalDateTimeLenient(String s) {
    String t = trimToNull(s);
    if (t == null) return null;

    // 1) без зоны: "2026-02-09T18:00:06"
    try { return LocalDateTime.parse(t); } catch (Exception ignore) {}

    // 2) с offset/Z: "2026-02-09T18:00:06Z" или "+03:00"
    try { return OffsetDateTime.parse(t).toLocalDateTime(); } catch (Exception ignore) {}

    throw new IllegalArgumentException("Invalid LocalDateTime: " + t);
  }

  private static LocalDate toLocalDateLenient(String s) {
    String t = trimToNull(s);
    if (t == null) return null;

    // 1) date: "2026-02-10"
    try { return LocalDate.parse(t); } catch (Exception ignore) {}

    // 2) datetime без зоны: "2026-02-10T00:00:00"
    try { return LocalDateTime.parse(t).toLocalDate(); } catch (Exception ignore) {}

    // 3) datetime c offset/Z
    try { return OffsetDateTime.parse(t).toLocalDate(); } catch (Exception ignore) {}

    throw new IllegalArgumentException("Invalid LocalDate: " + t);
  }
}


@JacksonXmlRootElement(localName = "FXRates")
@JsonIgnoreProperties(ignoreUnknown = true)
public class FxRateXmlDto {

  // элемент <IsPublic>...</IsPublic>
  @JacksonXmlProperty(localName = "IsPublic")
  @JsonDeserialize(converter = XmlConverters.TrimToNull.class)
  public String isPublic;

  // атрибуты FXRates ...
  @JacksonXmlProperty(isAttribute = true, localName = "FXRateSubType")
  @JsonDeserialize(converter = XmlConverters.UpperTrimToNull.class)
  public String fxRateSubType;

  @JacksonXmlProperty(isAttribute = true, localName = "Code1")
  @JsonDeserialize(converter = XmlConverters.UpperTrimToNull.class)
  public String code1;

  @JacksonXmlProperty(isAttribute = true, localName = "ISONum1")
  @JsonDeserialize(converter = XmlConverters.TrimToNull.class)
  public String isoNum1;

  @JacksonXmlProperty(isAttribute = true, localName = "Code2")
  @JsonDeserialize(converter = XmlConverters.NormalizeRubUpper.class)
  public String code2;

  @JacksonXmlProperty(isAttribute = true, localName = "ISONum2")
  @JsonDeserialize(converter = XmlConverters.TrimToNull.class)
  public String isoNum2;

  // если не сохраняешь — можешь вообще убрать поле
  @JacksonXmlProperty(isAttribute = true, localName = "PublicationDate")
  @JsonDeserialize(converter = XmlConverters.TrimToNull.class)
  public String publicationDate;

  @JacksonXmlProperty(isAttribute = true, localName = "UseDate")
  @JsonDeserialize(converter = XmlConverters.ToLocalDateLenient.class)
  public LocalDate useDate;

  @JacksonXmlProperty(isAttribute = true, localName = "LotSize")
  @JsonDeserialize(converter = XmlConverters.ToBigDecimal.class)
  public BigDecimal lotSize;

  @JacksonXmlProperty(isAttribute = true, localName = "Value")
  @JsonDeserialize(converter = XmlConverters.ToBigDecimal.class)
  public BigDecimal value;
}

@JacksonXmlRootElement(localName = "PutEODPriceN")
@JsonIgnoreProperties(ignoreUnknown = true)
public class PutEodPriceNDto {

  @JacksonXmlProperty(localName = "RQUID")
  @JsonDeserialize(converter = XmlConverters.ToUuid.class)
  public UUID rquid;

  @JacksonXmlProperty(localName = "RqTm")
  @JsonDeserialize(converter = XmlConverters.ToLocalDateTimeLenient.class)
  public LocalDateTime rqtm;

  @JacksonXmlElementWrapper(useWrapping = false)
  @JacksonXmlProperty(localName = "FXRates")
  public List<FxRateXmlDto> fxRates;
}

@JacksonXmlRootElement(localName = "PutEODPriceNf")
@JsonIgnoreProperties(ignoreUnknown = true)
public class PutEodPriceNfDto {

  @JacksonXmlProperty(localName = "RQUID")
  @JsonDeserialize(converter = XmlConverters.ToUuid.class)
  public UUID rquid;

  @JacksonXmlProperty(localName = "RqTm")
  @JsonDeserialize(converter = XmlConverters.ToLocalDateTimeLenient.class)
  public LocalDateTime rqtm;

  @JacksonXmlElementWrapper(useWrapping = false)
  @JacksonXmlProperty(localName = "FXRates")
  public List<FxRateXmlDto> fxRates;
}


```
