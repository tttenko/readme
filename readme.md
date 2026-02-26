```java/**
@Slf4j
@Component
@RequiredArgsConstructor
@ConditionalOnProperty(prefix = "app.kafka", name = "enabled", havingValue = "true", matchIfMissing = true)
public class CurrencyRateKafkaListener {

    private final CurrencyRateMessageProcessor processor;

    @KafkaListener(
        topics = "${app.kafka.currency-rate.topic}",
        containerFactory = "currencyRateContainerFactory"
    )
    public void handleMessage(ConsumerRecord<String, String> r) {
        // Факт получения сообщения
        log.info("KAFKA IN  topic={} partition={} offset={} key={} ts={} headers={} valueSize={}",
                r.topic(),
                r.partition(),
                r.offset(),
                r.key(),
                r.timestamp(),
                r.headers(),
                r.value() == null ? null : r.value().length()
        );

        long t0 = System.currentTimeMillis();
        try {
            processor.process(r);
            log.info("KAFKA OK  topic={} partition={} offset={} tookMs={}",
                    r.topic(), r.partition(), r.offset(), (System.currentTimeMillis() - t0));
        } catch (Exception e) {
            // Важно: чтобы точно было видно падение обработки
            log.error("KAFKA FAIL topic={} partition={} offset={} key={} err={}",
                    r.topic(), r.partition(), r.offset(), r.key(), e.toString(), e);
            throw e; // не глотаем, чтобы error handler отработал корректно
        }
    }
}
```
