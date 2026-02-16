```java
/**
 * Провайдер маршрутов Kafka для загрузки (ingest) курсов валют.
 * <p>
 * Формирует список {@link RouteDefinition} с привязкой ключа сообщения к DTO и обработчику,
 * используя настройки из {@link CurrencyRateKafkaProps} и сервис {@link FxRateIngestService}.
 */
@Component
@RequiredArgsConstructor
public class CurrencyRateIngestRoutesProvider implements RateKafkaRoutesProvider {

    private final FxRateIngestService ingestService;
    private final CurrencyRateKafkaProps props;

    /**
     * Возвращает список маршрутов обработки сообщений.
     * <p>
     * Каждый маршрут определяет ключ (алиас) сообщения, класс DTO для парсинга и consumer,
     * который выполняет обработку (ingest).
     *
     * @return неизменяемый список маршрутов обработки
     */
    @Override
    public List<RouteDefinition<?>> routes() { /* ... */ }
}

/**
 * Kafka-слушатель сообщений с курсами валют.
 * <p>
 * Получает записи из Kafka-топика, указанного в настройках приложения, и делегирует
 * обработку {@link XmlKafkaRecordProcessor}, используя {@link CurrencyRateRoutesRegistry}
 * для выбора нужного маршрута обработки.
 */
@Component
@RequiredArgsConstructor
@Log4j2
public class CurrencyRateKafkaListener {

    private final XmlKafkaRecordProcessor recordProcessor;
    private final CurrencyRateRoutesRegistry routesRegistry;

    /**
     * Обрабатывает входящее сообщение из Kafka.
     * <p>
     * Метод вызывается контейнером {@link org.springframework.kafka.annotation.KafkaListener}
     * и передаёт запись в {@link XmlKafkaRecordProcessor} вместе с реестром маршрутов.
     *
     * @param record запись Kafka с ключом и XML/строковым содержимым сообщения
     * @throws Exception если в процессе обработки произошла ошибка
     */
    @KafkaListener(
            topics = "${app.kafka.currency-rate.topic}",
            containerFactory = "currencyRateContainerFactory")
    public void handleMessage(final ConsumerRecord<String, String> record) throws Exception { /* ... */ }

}

/**
 * Резолвер ключа маршрутизации для сообщений с курсами валют.
 * <p>
 * Извлекает routeKey из XML-содержимого записи Kafka (как правило, по корневому тегу),
 * используя {@link FxRateXmlSupport}. В случае некорректного XML выбрасывает
 * {@link InvalidCurrencyRateXmlException} с диагностической информацией о записи.
 */
@Component
@RequiredArgsConstructor
public class CurrencyRateKeyResolver {

    private final FxRateXmlSupport fxRateXmlSupport;

    /**
     * Определяет ключ маршрута для входящей записи Kafka.
     *
     * @param record запись Kafka, содержащая XML в {@code value}
     * @return ключ маршрута (например, имя корневого тега XML)
     * @throws InvalidCurrencyRateXmlException если ключ невозможно извлечь из-за некорректного XML
     */
    public String resolve(ConsumerRecord<String, String> record) { /* ... */ }

}

/**
 * Возвращает актуальный (последний) курс для пары валют на указанную дату.
 * <p>
 * Ищет запись с максимальной {@code useDate}, которая строго меньше начала следующего дня
 * (т.е. курс «на дату»). Если курс не найден — возвращает пустой список.
 *
 * @param date дата, на которую требуется курс
 * @param isoNum1 числовой ISO-код базовой валюты
 * @param isoNum2 числовой ISO-код котируемой валюты
 * @return список из одного элемента с найденным курсом либо пустой список
 */
public List<CurrencyRateDto> getCurrencyRate(LocalDate date, String isoNum1, String isoNum2) { /* ... */ }

/**
 * Сервис загрузки (ingest) курсов валют/металлов из входящего XML-сообщения.
 * <p>
 * Выполняет валидацию обязательных полей, преобразует XML DTO в сущности и сохраняет их в БД через upsert.
 */
@Service
@RequiredArgsConstructor
@Log4j2
public class FxRateIngestService {

    private final FxRateRepository repo;
    private final FxRateEntityMapper fxRateEntityMapper;

    /**
     * Обрабатывает входящее сообщение {@link PutEodPriceNfDto} и сохраняет курсы в БД.
     *
     * @param message входящее XML-сообщение с курсами
     * @throws IllegalArgumentException       если сообщение равно {@code null}
     * @throws InvalidCurrencyRateXmlException если отсутствуют обязательные поля (RqUID/RqTm)
     */
    @Transactional
    public void ingest(PutEodPriceNfDto message) {
    }

    /**
     * Внутренняя обработка списка курсов: преобразование в сущности и сохранение через upsert.
     *
     * @param rquid идентификатор запроса
     * @param rgtm  время формирования сообщения
     * @param rates список курсов валют/металлов
     * @throws InvalidCurrencyRateXmlException если отсутствуют обязательные параметры
     */
    private void ingestInternal(String rquid, LocalDateTime rgtm, List<FxRateXmlDto> rates) {
    }
}

/**
 * Утилиты для безопасной предобработки и разбора XML-строки.
 * <p>
 * Используется для удаления BOM и извлечения имени корневого XML-тега (routeKey).
 */
@Component
public class FxRateXmlSupport {

    /**
     * Удаляет BOM (U+FEFF) и лишние пробелы из строки.
     *
     * @param s исходная строка
     * @return строка без BOM (или пустая строка, если {@code s == null})
     */
    public String stripBom(String s) {
    }

    /**
     * Извлекает имя корневого элемента XML (localName первого START_ELEMENT).
     *
     * @param xml XML в виде строки
     * @return имя корневого тега
     * @throws IllegalArgumentException если XML пустой/некорректный или корневой элемент не найден
     */
    public String extractRoot(String xml) {
    }
}

/**
 * Процессор Kafka-записей с XML-сообщениями.
 * <p>
 * Определяет ключ маршрута по содержимому XML, находит соответствующий {@link RouteDefinition}
 * в {@link RoutesRegistry}, десериализует XML в нужный DTO с помощью {@link XmlMapper}
 * и передаёт DTO в обработчик (consumer) маршрута.
 */
@Component
@RequiredArgsConstructor
public class XmlKafkaRecordProcessor {

    private final XmlMapper xmlMapper;
    private final CurrencyRateKeyResolver keyResolver;
    private final FxRateXmlSupport fxRateXmlSupport;

    /**
     * Обрабатывает запись Kafka: валидирует входные параметры, определяет routeKey,
     * извлекает маршрут из реестра и запускает типизированную обработку.
     *
     * @param record запись Kafka с XML в {@code value}
     * @param registry реестр маршрутов обработки
     * @throws Exception если возникла ошибка на этапе разрешения маршрута или обработки
     *                  (включая возможные исключения, прокинутые вызываемыми компонентами)
     */
    public void process(ConsumerRecord<String, String> record, RoutesRegistry registry) throws Exception { /* ... */ }

    /**
     * Типизированная обработка записи: удаляет BOM из XML (при наличии), парсит XML в DTO
     * класса маршрута и передаёт результат в consumer маршрута.
     * <p>
     * При ошибке парсинга выбрасывает {@link IllegalArgumentException} с диагностикой.
     *
     * @param record запись Kafka с исходным XML
     * @param route определение маршрута (ключ, класс DTO и обработчик)
     * @param <T> тип DTO маршрута
     * @throws IllegalArgumentException если XML не удалось распарсить в DTO маршрута
     */
    private <T> void processTyped(ConsumerRecord<String, String> record, RouteDefinition<T> route) { /* ... */ }
}
```
