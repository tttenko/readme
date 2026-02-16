```java
/**
 * Конфигурация Kafka для обработки сообщений с курсами валют.
 * <p>
 * Создаёт и настраивает фабрики Kafka consumer/producer, {@link KafkaTemplate},
 * а также фабрику контейнеров слушателей и обработчик ошибок.
 */
@Configuration
@EnableKafka
@EnableConfigurationProperties(CurrencyRateKafkaProps.class)
public class CurrencyRateKafkaConfig {

    /**
     * Создаёт {@link ConsumerFactory} для потребителей Kafka, читающих сообщения курсов валют.
     * <p>
     * Настраивает подключение к брокерам, группу, стратегию смещения, десериализаторы
     * и отключает автокоммит (подтверждение выполняется вручную контейнером).
     *
     * @param props свойства Kafka для курсов валют
     * @return фабрика consumer'ов Kafka
     */
    @Bean
    public ConsumerFactory<String, String> currencyRateConsumerFactory(CurrencyRateKafkaProps props) { /* ... */ }

    /**
     * Создаёт {@link ProducerFactory} для продюсеров Kafka, отправляющих сообщения курсов валют.
     * <p>
     * Настраивает подключение к брокерам и сериализаторы ключа/значения.
     *
     * @param props свойства Kafka для курсов валют
     * @return фабрика producer'ов Kafka
     */
    @Bean
    public ProducerFactory<String, String> currencyRateProducerFactory(CurrencyRateKafkaProps props) { /* ... */ }

    /**
     * Создаёт {@link KafkaTemplate} для отправки сообщений в Kafka.
     *
     * @param pf фабрика producer'ов
     * @return KafkaTemplate для строковых ключей и значений
     */
    @Bean
    public KafkaTemplate<String, String> currencyRateKafkaTemplate(ProducerFactory<String, String> pf) { /* ... */ }

    /**
     * Создаёт {@link ConcurrentKafkaListenerContainerFactory} для слушателей Kafka.
     * <p>
     * Настраивает конкурентность, режим подтверждения (ACK) на уровне записи (RECORD)
     * и общий обработчик ошибок.
     *
     * @param cf фабрика consumer'ов
     * @param props свойства Kafka для курсов валют
     * @return фабрика контейнеров слушателей с заданными настройками
     */
    @Bean(name = "currencyRateContainerFactory")
    public ConcurrentKafkaListenerContainerFactory<String, String> currencyRateContainerFactory(
            ConsumerFactory<String, String> cf,
            CurrencyRateKafkaProps props
    ) { /* ... */ }

    /**
     * Создаёт обработчик ошибок Kafka {@link DefaultErrorHandler}.
     * <p>
     * Настраивает политику повторов через {@link FixedBackOff}, список исключений без повторов
     * и поведение коммита смещений для "восстановленных" записей.
     *
     * @param props свойства Kafka для курсов валют
     * @return обработчик ошибок для контейнеров слушателей
     */
    @Bean
    public DefaultErrorHandler currencyRateErrorHandler(CurrencyRateKafkaProps props) { /* ... */ }
}


```
