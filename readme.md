```java/**
@Service
@RequiredArgsConstructor
public class CurrencyRateProcessor {

    private static final String EXPECTED_ROOT = "PutEODPriceNf";

    private final XmlMapper xmlMapper;
    private final FxRateXmlSupport fxRateXmlSupport;
    private final FxRateIngestService ingestService;

    public void process(ConsumerRecord<String, String> record) {
        String root = fxRateXmlSupport.extractRoot(record.value());
        if (!EXPECTED_ROOT.equals(root)) {
            throw new IllegalArgumentException("Unexpected XML root tag: " + root);
        }

        String xml = fxRateXmlSupport.stripBom(record.value());

        try {
            PutEodPriceNfDto dto = xmlMapper.readValue(xml, PutEodPriceNfDto.class);
            ingestService.ingest(dto);
        } catch (Exception e) {
            throw new IllegalArgumentException("Cannot parse PutEODPriceNf XML", e);
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
