```java
public String resolve(ConsumerRecord<String, String> record) {
        try {
            return FxRateXmlSupport.rootTag(record.value());
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException(
                "Cannot resolve routeKey from XML root tag. " +
                "topic=" + record.topic() +
                ", partition=" + record.partition() +
                ", offset=" + record.offset(),
                e
            );
        }
    }
```
