```java
class ApplicationConfigTest {

    private final ApplicationConfig config = new ApplicationConfig();

    @Test
    void webClientBeanCreated() {
        ExchangeStrategies strategies = ExchangeStrategies.builder()
                .codecs(c -> c.defaultCodecs().jackson2JsonDecoder(new Jackson2JsonDecoder()))
                .build();

        WebClient webClient = config.webClient(strategies);

        assertNotNull(webClient);

        // (опционально) Проверяем, что WebClient реально получил эти стратегии
        Object actualStrategies = ReflectionTestUtils.getField(webClient, "exchangeStrategies");
        assertSame(strategies, actualStrategies);
    }
}



```
