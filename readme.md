```java/**
@Component
@RequiredArgsConstructor
public class CurrencyRateKafkaListener {

    private final CurrencyRateMessageProcessor processor;

    @KafkaListener(
        topics = "${app.kafka.currency-rate.topic}",
        containerFactory = "currencyRateContainerFactory"
    )
    public void handleMessage(ConsumerRecord<String, String> record) {
        processor.process(record);
    }
}

@Service
@RequiredArgsConstructor
public class CurrencyRateMessageProcessor {

    private final PutEodPriceNfXmlParser parser;
    private final FxRateIngestService ingestService;

    public void process(ConsumerRecord<String, String> record) {
        if (record == null || record.value() == null || record.value().isBlank()) {
            throw new IllegalArgumentException("Kafka record/value is null or blank");
        }

        PutEodPriceNfDto dto = parser.parse(record);
        ingestService.ingest(dto);
    }
}

@Component
@RequiredArgsConstructor
public class PutEodPriceNfXmlParser {

    private final XmlMapper xmlMapper;
    private final FxRateXmlSupport fxRateXmlSupport;

    public PutEodPriceNfDto parse(ConsumerRecord<String, String> record) {
        String xml = fxRateXmlSupport.stripBom(record.value());

        try {
            return xmlMapper.readValue(xml, PutEodPriceNfDto.class);
        } catch (Exception e) {
            throw new InvalidCurrencyRateXmlException(
                "Не удалось распарсить PutEODPriceNf XML"
                    + ", topic=" + record.topic()
                    + ", partition=" + record.partition()
                    + ", offset=" + record.offset()
                    + ", key=" + record.key(),
                e
            );
        }
    }
}
```
