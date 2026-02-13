```java
@Bean
    public DefaultErrorHandler currencyRateErrorHandler(CurrencyRateKafkaProps props) {
        // retries для retryable исключений
        var backOff = new FixedBackOff(props.backoffMs(), props.maxAttempts());

        var errorHandler = new DefaultErrorHandler(backOff);

        // Битый XML / неизвестный роут / etc — НЕ ретраим, просто лог + пропуск
        errorHandler.addNotRetryableExceptions(
                InvalidCurrencyRateXmlException.class,
                IllegalArgumentException.class,
                UnsupportedOperationException.class
        );

        // важно: чтобы "пропущенное" сообщение коммитнулось и не читалось заново
        errorHandler.setCommitRecovered(true);

        // логирование
        errorHandler.setRetryListeners((record, ex, deliveryAttempt) -> {
            // deliveryAttempt: 1..N
            // тут можно разделять retry и финальный skip, но обычно достаточно общего лога
            org.slf4j.LoggerFactory.getLogger("kafka.currency-rate")
                    .warn("Kafka consume failed. attempt={}, topic={}, partition={}, offset={}, key={}, ex={}: {}",
                            deliveryAttempt,
                            record.topic(),
                            record.partition(),
                            record.offset(),
                            record.key(),
                            ex.getClass().getSimpleName(),
                            ex.getMessage(),
                            ex);
        });

        return errorHandler;
    }
```
