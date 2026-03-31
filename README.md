```java
@DataJpaTest(properties = {
        "spring.datasource.url=jdbc:h2:mem:test;MODE=PostgreSQL;DB_CLOSE_DELAY=-1",
        "spring.datasource.driver-class-name=org.h2.Driver",
        "spring.datasource.username=test",
        "spring.datasource.password=test",
        "app.kafka.enabled=false",
        "outbox.enabled=false"
})
```
