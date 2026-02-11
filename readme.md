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


@Component
@RequiredArgsConstructor
public class FxRateXmlParser {
  private static final Logger log = LoggerFactory.getLogger(FxRateXmlParser.class);

  // первый "нормальный" тег (не <?xml ... и не <!DOCTYPE/комментарий и не </close>)
  private static final Pattern ROOT_TAG = Pattern.compile("<\\s*(?!\\?|!|/)([^\\s/>]+)");

  private final XmlMapper xmlMapper;

  public ParsedFxRates parse(String xml) {
    if (xml == null || xml.isBlank()) {
      throw new IllegalArgumentException("XML is blank");
    }

    String cleaned = stripBom(xml);
    String root = extractRoot(cleaned);

    try {
      return switch (root) {
        case "PutEODPriceNF" -> {
          PutEodPriceNFDto doc = xmlMapper.readValue(cleaned, PutEodPriceNFDto.class);
          yield new ParsedFxRates(root, doc.rquid, doc.rqtm, doc.fxRates);
        }
        case "PutEODPriceN" -> {
          PutEodPriceNDto doc = xmlMapper.readValue(cleaned, PutEodPriceNDto.class);
          yield new ParsedFxRates(root, doc.rquid, doc.rqtm, doc.fxRates);
        }
        default -> throw new IllegalArgumentException("Unsupported XML root: " + root);
      };
    } catch (Exception e) {
      log.warn("Failed to parse FX XML. root={}, snippet={}", root, snippet(cleaned), e);
      throw (e instanceof IllegalArgumentException)
        ? (IllegalArgumentException) e
        : new IllegalArgumentException("Failed to parse FX XML. root=" + root, e);
    }
  }

  private static String stripBom(String s) {
    String t = s.trim();
    return t.startsWith("\uFEFF") ? t.substring(1).trim() : t;
  }

  private static String extractRoot(String xml) {
    Matcher m = ROOT_TAG.matcher(xml);
    if (!m.find()) return "UNKNOWN";

    String tag = m.group(1);
    int idx = tag.lastIndexOf(':'); // убираем namespace prefix если есть
    return idx >= 0 ? tag.substring(idx + 1) : tag;
  }

  private static String snippet(String s) {
    if (s == null) return "";
    String oneLine = s.replaceAll("\\s+", " ").trim();
    return oneLine.length() <= 200 ? oneLine : oneLine.substring(0, 200) + "...";
  }

  public record ParsedFxRates(String root, UUID rquid, LocalDateTime rqtm, List<FxRateXmlDto> fxRates) {}
}

```
