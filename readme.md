```java/**
try {
        processor.process(r);
        log.info("KAFKA OK  {}-{}@{}", r.topic(), r.partition(), r.offset());
    } catch (Exception e) {
        log.error("KAFKA FAIL {}-{}@{} {}", r.topic(), r.partition(), r.offset(), e.getMessage(), e);
        throw e;
    }
```
