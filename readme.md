```java

@Component
@RequiredArgsConstructor
public class FxRateXmlParser {
  private static final Logger log = LoggerFactory.getLogger(FxRateXmlParser.class);

  // первый "нормальный" тег (не <?xml ... и не <!DOCTYPE/комментарий и не </close>)
  private static final Pattern ROOT_TAG = Pattern.compile("<\\s*(?!\\?|!|/)([^\\s/>]+)");

  private static final Map<String, Class<? extends FxEnvelope>> ROOT_TO_DTO = Map.of(
    "PutEODPriceN",  PutEodPriceNDto.class,
    "PutEODPriceNF", PutEodPriceNFDto.class
  );

  private final XmlMapper xmlMapper;

  /** Унифицированный результат, root в бизнес-логике не нужен */
  public FxMessage parse(String xml) {
    if (xml == null || xml.isBlank()) throw new IllegalArgumentException("XML is blank");

    String cleaned = stripBom(xml);
    String root = extractRoot(cleaned);

    Class<? extends FxEnvelope> dtoClass = ROOT_TO_DTO.get(root);
    if (dtoClass == null) {
      throw new IllegalArgumentException("Unsupported XML root: " + root);
    }

    try {
      FxEnvelope doc = xmlMapper.readValue(cleaned, dtoClass);
      return new FxMessage(doc.getRquid(), doc.getRqtm(), doc.getFxRates());
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

  /** То, что реально нужно дальше в сервисе */
  public record FxMessage(UUID rquid, LocalDateTime rqtm, List<FxRateXmlDto> fxRates) {}

  /**
   * Общий контракт для "N" и "NF" dto.
   * Можно сделать interface и реализовать в обоих DTO без дублирования бизнес-логики.
   */
  public interface FxEnvelope {
    UUID getRquid();
    LocalDateTime getRqtm();
    List<FxRateXmlDto> getFxRates();
  }
}
```
