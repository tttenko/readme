```java
log.info(
                "Kafka message received: topic={}, partition={}, offset={}, timestamp={}, key={}, valueSize={}",
                record.topic(),
                record.partition(),
                record.offset(),
                record.timestamp(),
                record.key(),
                record.value() == null ? null : record.value().length()
        );

        // Если надо видеть часть XML/JSON — аккуратно обрежем
        if (log.isDebugEnabled()) {
            log.debug("Kafka message payload (trimmed): {}", trim(record.value()));
        }

        try {
            recordProcessor.process(record, routesRegistry);

            log.info(
                    "Kafka message processed OK: topic={}, partition={}, offset={}, key={}",
                    record.topic(), record.partition(), record.offset(), record.key()
            );
        } catch (Exception e) {
            log.error(
                    "Kafka message processing FAILED: topic={}, partition={}, offset={}, key={}",
                    record.topic(), record.partition(), record.offset(), record.key(),
                    e
            );
            throw e; // важно пробросить, чтобы поведение retry/DLT осталось таким, как настроено
        }
    }

    private static String trim(String s) {
        if (s == null) return null;
        if (s.length() <= MAX_VALUE_LOG_LEN) return s;
        return s.substring(0, MAX_VALUE_LOG_LEN) + "...(trimmed, len=" + s.length() + ")";
    }

```
