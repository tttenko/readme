```java
app:
  kafka:
    currency-rate:
      topic: rqCreateCurrencyRate
      group-id: cg_currency_rate_ingest
      servers: "tvldq-bsiba0605.delta.sbrf.ru:9093,tvldq-bsiba0606.delta.sbrf.ru:9093"
      auto-offset-reset: earliest
      concurrency: 1


@ConfigurationProperties(prefix = "app.kafka.currency-rate")
public record CurrencyRateKafkaProps(
    String topic,
    String groupId,
    String servers,
    String autoOffsetReset,
    int concurrency
) {}

@Configuration
@EnableKafka
@EnableConfigurationProperties(CurrencyRateKafkaProps.class)
/**
 * Kafka-конфигурация для чтения XML сообщений с курсами/ценами и обработки ошибок.
 *
 * <p>Что настраивает:
 * <ul>
 *   <li>ConsumerFactory для чтения сообщений как String</li>
 *   <li>ProducerFactory + KafkaTemplate для публикации проблемных сообщений в DLT</li>
 *   <li>Listener container factory с ручным коммитом оффсетов</li>
 * </ul>
 *
 * <p>Стратегия ошибок:
 * <ul>
 *   <li>3 ретрая с паузой 1 секунда</li>
 *   <li>после ретраев сообщение отправляется в DLT (&lt;topic&gt;.DLT)</li>
 *   <li>{@link IllegalArgumentException} считается неретраибельной и сразу отправляется в DLT</li>
 *   <li>после отправки в DLT оффсет коммитится, чтобы не зациклиться на “битом” сообщении</li>
 * </ul>
 */
public class CurrencyRateKafkaConfig {

  /**
   * ConsumerFactory для чтения сообщений из Kafka.
   * Автокоммит выключен: оффсет фиксируется только после успешной обработки.
   */
  @Bean
  public ConsumerFactory<String, String> currencyRateConsumerFactory(CurrencyRateKafkaProps props) {
    Map<String, Object> cfg = new HashMap<>();
    cfg.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, props.servers());
    cfg.put(ConsumerConfig.GROUP_ID_CONFIG, props.groupId());
    cfg.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, props.autoOffsetReset());
    cfg.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    cfg.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    cfg.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
    return new DefaultKafkaConsumerFactory<>(cfg);
  }

  /**
   * ProducerFactory для отправки сообщений в DLT (dead-letter topic).
   */
  @Bean
  public ProducerFactory<String, String> currencyRateProducerFactory(CurrencyRateKafkaProps props) {
    Map<String, Object> cfg = new HashMap<>();
    cfg.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, props.servers());
    cfg.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    cfg.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    return new DefaultKafkaProducerFactory<>(cfg);
  }

  /**
   * KafkaTemplate для публикации сообщений в DLT.
   */
  @Bean
  public KafkaTemplate<String, String> currencyRateKafkaTemplate(ProducerFactory<String, String> pf) {
    return new KafkaTemplate<>(pf);
  }

  /**
   * Фабрика listener-контейнеров для @KafkaListener.
   * Настроена на:
   * <ul>
   *   <li>AckMode.RECORD — коммит оффсета после обработки каждой записи</li>
   *   <li>DLT на &lt;topic&gt;.DLT</li>
   *   <li>3 ретрая с паузой 1 секунда</li>
   * </ul>
   */
  @Bean(name = "currencyRateContainerFactory")
  public ConcurrentKafkaListenerContainerFactory<String, String> currencyRateContainerFactory(
      ConsumerFactory<String, String> cf,
      KafkaTemplate<String, String> kafkaTemplate,
      CurrencyRateKafkaProps props
  ) {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
    factory.setConsumerFactory(cf);
    factory.setConcurrency(props.concurrency());
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD);

    // DLT: <topic>.DLT, retry 3 раза, потом в DLT
    var recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);
    var errorHandler = new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3L));

    // Битые сообщения/валидация -> сразу в DLT
    errorHandler.addNotRetryableExceptions(IllegalArgumentException.class);

    // Коммитить оффсет после отправки в DLT, чтобы не зациклиться
    errorHandler.setCommitRecovered(true);

    factory.setCommonErrorHandler(errorHandler);
    return factory;
  }
}

@Configuration
public class XmlConfig {
  @Bean
  public XmlMapper xmlMapper() {
    XmlMapper mapper = new XmlMapper();
    mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
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
  public String publicationDate;

  @JacksonXmlProperty(isAttribute = true, localName = "UseDate")
  public String useDate;

  @JacksonXmlProperty(isAttribute = true, localName = "LotSize")
  public String lotSize;

  @JacksonXmlProperty(isAttribute = true, localName = "Value")
  public String value;
}


@JacksonXmlRootElement(localName = "PutEODPriceN")
@JsonIgnoreProperties(ignoreUnknown = true)
public class PutEodPriceNDto {

  @JacksonXmlProperty(localName = "RQUID")
  public String rquid;

  @JacksonXmlProperty(localName = "RqTm")
  public String rqtm;

  @JacksonXmlElementWrapper(useWrapping = false)
  @JacksonXmlProperty(localName = "FXRates")
  public List<FxRateXmlDto> fxRates;
}


@JacksonXmlRootElement(localName = "PutEODPriceNf")
@JsonIgnoreProperties(ignoreUnknown = true)
public class PutEodPriceNfDto {

  @JacksonXmlProperty(localName = "RQUID")
  public String rquid;

  @JacksonXmlProperty(localName = "RqTm")
  public String rqtm;

  @JacksonXmlElementWrapper(useWrapping = false)
  @JacksonXmlProperty(localName = "FXRates")
  public List<FxRateXmlDto> fxRates;
}


@Entity
@Table(name = "fx_rate")
public class FxRateEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(name = "rquid", nullable = false)
  private UUID rquid;

  @Column(name = "rqtm", nullable = false)
  private LocalDateTime rqtm;

  @Column(name = "fxratesubtype", nullable = false, length = 8)
  private String fxRateSubType;

  @Column(name = "code1", nullable = false, length = 16)
  private String code1;

  @Column(name = "isonum1", length = 16)
  private String isoNum1;

  @Column(name = "code2", nullable = false, length = 16)
  private String code2;

  @Column(name = "isonum2", length = 16)
  private String isoNum2;

  @Column(name = "use_date", nullable = false)
  private LocalDate useDate; // DATE

  @Column(name = "lot_size", precision = 19, scale = 8)
  private BigDecimal lotSize;

  @Column(name = "value", precision = 19, scale = 8)
  private BigDecimal value;

  @Column(name = "is_public", length = 16)
  private String isPublic;

  // getters/setters (сократил)
  public UUID getRquid() { return rquid; }
  public void setRquid(UUID rquid) { this.rquid = rquid; }
  public LocalDateTime getRqtm() { return rqtm; }
  public void setRqtm(LocalDateTime rqtm) { this.rqtm = rqtm; }
  public String getFxRateSubType() { return fxRateSubType; }
  public void setFxRateSubType(String fxRateSubType) { this.fxRateSubType = fxRateSubType; }
  public String getCode1() { return code1; }
  public void setCode1(String code1) { this.code1 = code1; }
  public String getIsoNum1() { return isoNum1; }
  public void setIsoNum1(String isoNum1) { this.isoNum1 = isoNum1; }
  public String getCode2() { return code2; }
  public void setCode2(String code2) { this.code2 = code2; }
  public String getIsoNum2() { return isoNum2; }
  public void setIsoNum2(String isoNum2) { this.isoNum2 = isoNum2; }
  public LocalDate getUseDate() { return useDate; }
  public void setUseDate(LocalDate useDate) { this.useDate = useDate; }
  public BigDecimal getLotSize() { return lotSize; }
  public void setLotSize(BigDecimal lotSize) { this.lotSize = lotSize; }
  public BigDecimal getValue() { return value; }
  public void setValue(BigDecimal value) { this.value = value; }
  public String getIsPublic() { return isPublic; }
  public void setIsPublic(String isPublic) { this.isPublic = isPublic; }
}


public interface FxRateRepository extends JpaRepository<FxRateEntity, Long> {

  @Modifying(clearAutomatically = true, flushAutomatically = true)
  @Transactional
  @Query(value = """
    INSERT INTO fx_rate (
      rquid, rqtm, fxratesubtype, code1, isonum1, code2, isonum2, use_date, lot_size, value, is_public
    ) VALUES (
      :rquid, :rqtm, :subType, :code1, :isoNum1, :code2, :isoNum2, :useDate, :lotSize, :value, :isPublic
    )
    ON CONFLICT (fxratesubtype, code1, code2, use_date)
    DO UPDATE SET
      rquid = EXCLUDED.rquid,
      rqtm  = EXCLUDED.rqtm,
      isonum1 = EXCLUDED.isonum1,
      isonum2 = EXCLUDED.isonum2,
      lot_size = EXCLUDED.lot_size,
      value = EXCLUDED.value,
      is_public = EXCLUDED.is_public
    """, nativeQuery = true)
  int upsert(
      @Param("rquid") UUID rquid,
      @Param("rqtm") LocalDateTime rqtm,
      @Param("subType") String subType,
      @Param("code1") String code1,
      @Param("isoNum1") String isoNum1,
      @Param("code2") String code2,
      @Param("isoNum2") String isoNum2,
      @Param("useDate") LocalDate useDate,
      @Param("lotSize") BigDecimal lotSize,
      @Param("value") BigDecimal value,
      @Param("isPublic") String isPublic
  );
}


@Service
public class FxRateXmlParser {

  private static final Logger log = LoggerFactory.getLogger(FxRateXmlParser.class);
  private static final Pattern ROOT_TAG = Pattern.compile("<\\s*([^!?][^\\s>/]+)");

  private final XmlMapper xmlMapper;

  public FxRateXmlParser(XmlMapper xmlMapper) {
    this.xmlMapper = xmlMapper;
  }

  public ParsedFxRates parse(String xml) {
    String cleaned = stripBom(xml);
    String root = extractRoot(cleaned);

    try {
      return switch (root) {
        case "PutEODPriceNf" -> {
          PutEodPriceNfDto doc = xmlMapper.readValue(cleaned, PutEodPriceNfDto.class);
          yield new ParsedFxRates(doc.rquid, doc.rqtm, doc.fxRates);
        }
        case "PutEODPriceN" -> {
          PutEodPriceNDto doc = xmlMapper.readValue(cleaned, PutEodPriceNDto.class);
          yield new ParsedFxRates(doc.rquid, doc.rqtm, doc.fxRates);
        }
        default -> throw new IllegalArgumentException("Unsupported XML root: " + root);
      };
    } catch (Exception e) {
      log.warn("Failed to parse FX XML. root={}, snippet={}", root, snippet(cleaned), e);
      throw new IllegalArgumentException("Failed to parse FX XML. root=" + root, e);
    }
  }

  private static String stripBom(String s) {
    if (s == null) return "";
    String t = s.trim();
    return t.startsWith("\uFEFF") ? t.substring(1).trim() : t;
  }

  private static String extractRoot(String xml) {
    var m = ROOT_TAG.matcher(xml);
    if (!m.find()) return "UNKNOWN";
    String tag = m.group(1);
    int idx = tag.indexOf(':'); // ns:PutEODPriceN
    return idx >= 0 ? tag.substring(idx + 1) : tag;
  }

  private static String snippet(String s) {
    String oneLine = s.replaceAll("\\s+", " ").trim();
    return oneLine.length() <= 200 ? oneLine : oneLine.substring(0, 200) + "...";
  }

  public record ParsedFxRates(String rquid, String rqtm, List<FxRateXmlDto> fxRates) {}
}


public final class FxRateMapper {
  private FxRateMapper() {}

  public static UUID uuidOrThrow(String s, String field) {
    if (s == null || s.isBlank()) throw new IllegalArgumentException("Missing " + field);
    try { return UUID.fromString(s.trim()); }
    catch (Exception e) { throw new IllegalArgumentException("Invalid " + field + ": " + s, e); }
  }

  public static LocalDateTime dateTimeOrThrow(String s, String field) {
    LocalDateTime dt = parseDateTime(s);
    if (dt == null) throw new IllegalArgumentException("Invalid " + field + ": " + s);
    return dt;
  }

  public static LocalDate useDateOrThrow(String s) {
    LocalDateTime dt = parseDateTime(s);
    if (dt == null) throw new IllegalArgumentException("Invalid UseDate: " + s);
    return dt.toLocalDate();
  }

  private static LocalDateTime parseDateTime(String s) {
    if (s == null || s.isBlank()) return null;
    String t = s.trim();
    try { return LocalDateTime.parse(t); }
    catch (Exception ignore) {
      try { return OffsetDateTime.parse(t).toLocalDateTime(); }
      catch (Exception ignore2) { return null; }
    }
  }

  public static BigDecimal decimal(String s) {
    if (s == null || s.isBlank()) return null;
    try { return new BigDecimal(s.trim()); }
    catch (Exception e) { return null; }
  }

  public static String upperTrim(String s) {
    if (s == null) return null;
    String t = s.trim();
    return t.isEmpty() ? null : t.toUpperCase(Locale.ROOT);
  }

  public static String trimToNull(String s) {
    if (s == null) return null;
    String t = s.trim();
    return t.isEmpty() ? null : t;
  }

  public static String normalizeRub(String code2) {
    if (code2 == null) return null;
    return "RUR".equals(code2) ? "RUB" : code2;
  }
}


@Service
public class FxRateIngestService {

  private static final Logger log = LoggerFactory.getLogger(FxRateIngestService.class);

  private final FxRateXmlParser parser;
  private final FxRateRepository repo;

  public FxRateIngestService(FxRateXmlParser parser, FxRateRepository repo) {
    this.parser = parser;
    this.repo = repo;
  }

  @Transactional
  public int ingest(String xml) {
    var parsed = parser.parse(xml);

    UUID rquid = FxRateMapper.uuidOrThrow(parsed.rquid(), "RQUID");
    LocalDateTime rqtm = FxRateMapper.dateTimeOrThrow(parsed.rqtm(), "RqTm");

    List<FxRateXmlDto> rates = parsed.fxRates();
    if (rates == null || rates.isEmpty()) {
      log.info("No FXRates in message. rquid={}", rquid);
      return 0;
    }

    int processed = 0;

    for (FxRateXmlDto x : rates) {
      if (x == null) continue;

      String subType = FxRateMapper.upperTrim(x.fxRateSubType);
      String code1 = FxRateMapper.upperTrim(x.code1);
      String code2 = FxRateMapper.normalizeRub(FxRateMapper.upperTrim(x.code2));
      LocalDate useDate = FxRateMapper.useDateOrThrow(x.useDate);

      if (subType == null || code1 == null || code2 == null) {
        log.debug("Skip record with missing key. rquid={}, subType={}, code1={}, code2={}, useDate={}",
            rquid, subType, code1, code2, useDate);
        continue;
      }

      BigDecimal lotSize = FxRateMapper.decimal(x.lotSize);
      BigDecimal value = FxRateMapper.decimal(x.value);

      // если lotSize=0 — это плохие данные, лучше в DLT
      if (lotSize != null && BigDecimal.ZERO.compareTo(lotSize) == 0) {
        throw new IllegalArgumentException("LotSize=0 for " + code1 + "/" + code2 + " useDate=" + useDate + " rquid=" + rquid);
      }

      repo.upsert(
          rquid, rqtm,
          subType,
          code1, FxRateMapper.trimToNull(x.isoNum1),
          code2, FxRateMapper.trimToNull(x.isoNum2),
          useDate,
          lotSize, value,
          FxRateMapper.trimToNull(x.isPublic)
      );

      processed++;
    }

    log.info("Ingest done. rquid={}, processed={}", rquid, processed);
    return processed;
  }
}


@Component
public class CurrencyRateKafkaListener {

  private final FxRateIngestService ingestService;

  public CurrencyRateKafkaListener(FxRateIngestService ingestService) {
    this.ingestService = ingestService;
  }

  @KafkaListener(
      topics = "${app.kafka.currency-rate.topic}",
      containerFactory = "currencyRateContainerFactory"
  )
  public void onMessage(String xml) {
    ingestService.ingest(xml);
  }
}



```
