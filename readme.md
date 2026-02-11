```java

@Getter
@AllArgsConstructor
public class RouteDefinition<T> {

  @NonNull
  private final String key;

  @NonNull
  private final Class<T> dtoClass;

  @NonNull
  private final Consumer<T> consumer;

  public static <T> RouteDefinition<T> route(
      @NonNull final String key,
      @NonNull final Class<T> dtoClass,
      @NonNull final Consumer<T> consumer
  ) {
    return new RouteDefinition<>(key, dtoClass, consumer);
  }
}

public interface KafkaRoutesProvider {
  List<RouteDefinition<?>> routes();
}

public class RoutesRegistry {

  private final Map<String, RouteDefinition<?>> routesByKey;

  public RoutesRegistry(@NonNull final List<? extends KafkaRoutesProvider> providers) {
    this.routesByKey = providers.stream()
        .flatMap(p -> p.routes().stream())
        .collect(Collectors.toUnmodifiableMap(
            RouteDefinition::getKey,
            Function.identity(),
            (a, b) -> {
              throw new IllegalStateException("Дублирующийся маршрут для key=" + a.getKey());
            }
        ));
  }

  @SuppressWarnings("unchecked")
  public <T> RouteDefinition<T> getRequired(@NonNull final String key) {
    final RouteDefinition<?> route = routesByKey.get(key);
    if (route == null) {
      throw new UnsupportedOperationException("Нет маршрута для key=" + key);
    }
    return (RouteDefinition<T>) route;
  }
}

// CurrencyRateKafkaRoutesProvider.java
public interface CurrencyRateKafkaRoutesProvider extends KafkaRoutesProvider {
  // marker interface
}

@Component
public class CurrencyRateRoutesRegistry extends RoutesRegistry {

  public CurrencyRateRoutesRegistry(List<CurrencyRateKafkaRoutesProvider> providers) {
    super(providers);
  }
}

public final class FxRateXmlSupport {

  private FxRateXmlSupport() {}

  // первый тег документа (root)
  private static final Pattern ROOT_TAG = Pattern.compile("<\\s*([A-Za-z_][^\\s>/]*)");

  public static String stripBom(String s) {
    if (s == null) return "";
    String t = s.trim();
    return t.startsWith("\uFEFF") ? t.substring(1).trim() : t;
  }

  public static String extractRoot(String xml) {
    if (xml == null) return "UNKNOWN";
    var m = ROOT_TAG.matcher(xml);
    if (!m.find()) return "UNKNOWN";
    String tag = m.group(1);
    int idx = tag.indexOf(':');
    return (idx >= 0) ? tag.substring(idx + 1) : tag; // убираем ns:
  }

  public static String snippet(String xml) {
    if (xml == null) return "";
    String oneLine = xml.replaceAll("\\s+", " ").trim();
    return oneLine.length() <= 200 ? oneLine : oneLine.substring(0, 200) + "...";
  }
}

@ConfigurationProperties(prefix = "app.kafka.currency-rate")
public record CurrencyRateKafkaProps(
    String topic,
    String groupId,
    String servers,
    String autoOffsetReset,
    int concurrency,
    Map<String, String> keyAliases
) {
  public CurrencyRateKafkaProps {
    if (autoOffsetReset == null || autoOffsetReset.isBlank()) {
      autoOffsetReset = "earliest";
    }
    if (keyAliases == null) {
      keyAliases = Map.of();
    }
  }
}

@Configuration
@EnableKafka
@EnableConfigurationProperties(CurrencyRateKafkaProps.class)
public class CurrencyRateKafkaConfig {

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

  @Bean
  public ProducerFactory<String, String> currencyRateProducerFactory(CurrencyRateKafkaProps props) {
    Map<String, Object> cfg = new HashMap<>();
    cfg.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, props.servers());
    cfg.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    cfg.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    return new DefaultKafkaProducerFactory<>(cfg);
  }

  @Bean
  public KafkaTemplate<String, String> currencyRateKafkaTemplate(ProducerFactory<String, String> pf) {
    return new KafkaTemplate<>(pf);
  }

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

    var recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);
    var errorHandler = new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3L));
    errorHandler.addNotRetryableExceptions(IllegalArgumentException.class, UnsupportedOperationException.class);
    errorHandler.setCommitRecovered(true);

    factory.setCommonErrorHandler(errorHandler);
    return factory;
  }
}

@Component
public class CurrencyRateKeyResolver {

  private final CurrencyRateKafkaProps props;

  public CurrencyRateKeyResolver(CurrencyRateKafkaProps props) {
    this.props = props;
  }

  public String resolve(ConsumerRecord<String, String> record) {
    // 1) пробуем key
    String key = record.key();
    if (key != null && !key.isBlank()) {
      String normalized = key.trim();
      String alias = props.keyAliases().get(normalized);
      return (alias != null) ? alias : normalized;
    }

    // 2) fallback: корневой тег XML
    String xml = FxRateXmlSupport.stripBom(record.value());
    String root = FxRateXmlSupport.extractRoot(xml);

    String alias = props.keyAliases().get(root);
    return (alias != null) ? alias : root;
  }
}

@Component
public class XmlKafkaRecordProcessor {

  private final XmlMapper xmlMapper;
  private final CurrencyRateKeyResolver keyResolver;

  // validator опционален (если нет starter-validation, просто не валидируем)
  @Autowired(required = false)
  private Validator validator;

  public XmlKafkaRecordProcessor(XmlMapper xmlMapper, CurrencyRateKeyResolver keyResolver) {
    this.xmlMapper = xmlMapper;
    this.keyResolver = keyResolver;
  }

  public void process(ConsumerRecord<String, String> record, RoutesRegistry registry) throws Exception {
    String routeKey = keyResolver.resolve(record);
    RouteDefinition<?> route = registry.getRequired(routeKey);
    processTyped(record, route);
  }

  private <T> void processTyped(ConsumerRecord<String, String> record, RouteDefinition<T> route) {
    final String xml = FxRateXmlSupport.stripBom(record.value());

    final T dto;
    try {
      dto = xmlMapper.readValue(xml, route.getDtoClass());
    } catch (Exception e) {
      throw new IllegalArgumentException(
          "Не удалось распарсить XML для routeKey=" + route.getKey()
              + ", record.key=" + record.key()
              + ", snippet=" + FxRateXmlSupport.snippet(xml),
          e
      );
    }

    validate(dto);
    route.getConsumer().accept(dto);
  }

  private void validate(Object dto) {
    if (validator == null || dto == null) return;
    Set<ConstraintViolation<Object>> violations = validator.validate(dto);
    if (!violations.isEmpty()) {
      throw new ConstraintViolationException(violations);
    }
  }
}

@Component
public class CurrencyRateIngestRoutesProvider implements CurrencyRateKafkaRoutesProvider {

  private final FxRateIngestService ingestService;

  public CurrencyRateIngestRoutesProvider(FxRateIngestService ingestService) {
    this.ingestService = ingestService;
  }

  @Override
  public List<RouteDefinition<?>> routes() {
    return List.of(
        RouteDefinition.route("PutEODPriceN", PutEodPriceNDto.class, dto -> ingestService.ingest(dto)),
        RouteDefinition.route("PutEODPriceNf", PutEodPriceNfDto.class, dto -> ingestService.ingest(dto))
    );
  }
}

@Component
public class CurrencyRateKafkaListener {

  private final XmlKafkaRecordProcessor recordProcessor;
  private final CurrencyRateRoutesRegistry routesRegistry;

  public CurrencyRateKafkaListener(XmlKafkaRecordProcessor recordProcessor,
                                   CurrencyRateRoutesRegistry routesRegistry) {
    this.recordProcessor = recordProcessor;
    this.routesRegistry = routesRegistry;
  }

  @KafkaListener(
      topics = "${app.kafka.currency-rate.topic}",
      containerFactory = "currencyRateContainerFactory"
  )
  public void handleMessage(final ConsumerRecord<String, String> record) throws Exception {
    recordProcessor.process(record, routesRegistry);
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

@Service
public class FxRateIngestService {

  private static final Logger log = LoggerFactory.getLogger(FxRateIngestService.class);

  private final FxRateRepository repo;

  public FxRateIngestService(FxRateRepository repo) {
    this.repo = repo;
  }

  @Transactional
  public int ingest(PutEodPriceNDto doc) {
    if (doc == null) throw new IllegalArgumentException("doc is null");
    return ingestInternal(doc.rquid, doc.rqtm, doc.fxRates);
  }

  @Transactional
  public int ingest(PutEodPriceNfDto doc) {
    if (doc == null) throw new IllegalArgumentException("doc is null");
    return ingestInternal(doc.rquid, doc.rqtm, doc.fxRates);
  }

  private int ingestInternal(String rquidStr, String rqtmStr, List<FxRateXmlDto> rates) {
    UUID rquid = FxRateMapper.uuidOrThrow(rquidStr, "RQUID");
    LocalDateTime rqtm = FxRateMapper.dateTimeOrThrow(rqtmStr, "RqTm");

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



```
