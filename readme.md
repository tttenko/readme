```java/**
@Service
@RequiredArgsConstructor
public class CurrencyRateProcessor {

    private static final String EXPECTED_ROOT = "PutEODPriceNf";

    private final XmlMapper xmlMapper;
    private final FxRateXmlSupport fxRateXmlSupport;
    private final FxRateIngestService ingestService;

    public void process(ConsumerRecord<String, String> record) {
        if (record == null || record.value() == null) {
            throw new IllegalArgumentException("Kafka record/value is null");
        }

        String root;
        try {
            root = fxRateXmlSupport.extractRoot(record.value());
        } catch (IllegalArgumentException e) {
            throw new InvalidCurrencyRateXmlException(
                "Cannot resolve routeKey from XML root tag. "
                    + ", topic=" + record.topic()
                    + ", partition=" + record.partition()
                    + ", offset=" + record.offset(),
                e
            );
        }

        if (!EXPECTED_ROOT.equals(root)) {
            throw new InvalidCurrencyRateXmlException(
                "Unexpected XML root tag: " + root
                    + ", expected=" + EXPECTED_ROOT
                    + ", topic=" + record.topic()
                    + ", partition=" + record.partition()
                    + ", offset=" + record.offset()
            );
        }

        String xml = fxRateXmlSupport.stripBom(record.value());

        try {
            PutEodPriceNfDto dto = xmlMapper.readValue(xml, PutEodPriceNfDto.class);
            ingestService.ingest(dto);
        } catch (Exception e) {
            throw new InvalidCurrencyRateXmlException(
                "Не удалось распарсить XML " + EXPECTED_ROOT
                    + ", topic=" + record.topic()
                    + ", partition=" + record.partition()
                    + ", offset=" + record.offset()
                    + ", key=" + record.key(),
                e
            );
        }
    }
}

@Component
@RequiredArgsConstructor
public class CurrencyRateKafkaListener {
    private final CurrencyRateProcessor processor;

    @KafkaListener(
        topics = "${app.kafka.currency-rate.topic}",
        containerFactory = "currencyRateContainerFactory"
    )
    public void handleMessage(ConsumerRecord<String, String> record) {
        processor.process(record);
    }
}

```
