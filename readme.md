```java/**
if (record == null) {
        throw new IllegalArgumentException("Kafka record is null");
    }

    if (record.value() == null || record.value().isBlank()) {
        throw new IllegalArgumentException(
            "Kafka record value is null/blank"
                + ", topic=" + record.topic()
                + ", partition=" + record.partition()
                + ", offset=" + record.offset()
                + ", key=" + record.key()
        );
    }
```
