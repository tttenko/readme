```java/**
/**
 * Обрабатывает входящие сообщения о курсах валют из Kafka:
 * проверяет запись, парсит XML в DTO и передаёт данные в сервис сохранения.
 */
@Service
@RequiredArgsConstructor
public class CurrencyRateMessageProcessor {

    private final PutEodPriceNfXmlParser parser;
    private final FxRateIngestService ingestService;

    /**
     * Обрабатывает одну запись Kafka: валидирует наличие значения, парсит XML и запускает ingest.
     *
     * @param record запись Kafka с XML в {@link ConsumerRecord#value()}
     * @throws IllegalArgumentException если запись {@code record} равна {@code null} или {@code value} пустое
     * @throws InvalidCurrencyRateXmlException если не удалось распарсить XML в DTO
     */
    public void process(ConsumerRecord<String, String> record) {
        // ...
    }
}

/**
 * Парсер входящего XML-сообщения {@code PutEODPriceNf} в DTO {@link PutEodPriceNfDto}.
 */
@Component
@RequiredArgsConstructor
public class PutEodPriceNfXmlParser {

    private final XmlMapper xmlMapper;
    private final FxRateXmlSupport fxRateXmlSupport;

    /**
     * Десериализует XML из {@link ConsumerRecord#value()} в {@link PutEodPriceNfDto}.
     *
     * @param record запись Kafka с XML в {@link ConsumerRecord#value()}
     * @return распарсенный DTO {@link PutEodPriceNfDto}
     * @throws InvalidCurrencyRateXmlException если XML не удалось распарсить
     */
    public PutEodPriceNfDto parse(ConsumerRecord<String, String> record) {
        // ...
    }
}
```
