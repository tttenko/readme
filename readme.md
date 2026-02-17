```java/**
@SpringBootTest(classes = CurrencyRateKafkaPropsTest.TestConfig.class)
@TestPropertySource(properties = {
        "app.kafka.currency-rate.topic=topic-1",
        "app.kafka.currency-rate.group-id=group-1",
        "app.kafka.currency-rate.servers=localhost:9092",
        "app.kafka.currency-rate.auto-offset-reset=earliest",
        "app.kafka.currency-rate.concurrency=3",
        "app.kafka.currency-rate.backoff-ms=1500",
        "app.kafka.currency-rate.max-attempts=7",
        "app.kafka.currency-rate.key-aliases=alias1,alias2"
})
class CurrencyRateKafkaPropsTest {

    @Autowired
    private CurrencyRateKafkaProps props;

    @Test
    void shouldBindConfigurationProperties() {
        assertThat(props.topic()).isEqualTo("topic-1");
        assertThat(props.groupId()).isEqualTo("group-1");
        assertThat(props.servers()).isEqualTo("localhost:9092");
        assertThat(props.autoOffsetReset()).isEqualTo("earliest");
        assertThat(props.concurrency()).isEqualTo(3);
        assertThat(props.backoffMs()).isEqualTo(1500L);
        assertThat(props.maxAttempts()).isEqualTo(7L);
        assertThat(props.keyAliases()).isEqualTo("alias1,alias2");
    }

    @Configuration
    @EnableConfigurationProperties(CurrencyRateKafkaProps.class)
    static class TestConfig {
    }
}

class FxRateEntityMapperTest {

    private final FxRateEntityMapper mapper = new FxRateEntityMapper() {
        @Override
        public FxRateEntity toEntity(String requestUid, LocalDateTime requestTime, FxRateXmlDto fxRateXmlDto) {
            return null; // нам в этих тестах не нужен mapstruct mapping
        }
    };

    @Test
    void toZoned_whenNull_thenReturnNull() {
        assertThat(mapper.toZoned(null)).isNull();
    }

    @Test
    void toZoned_whenNotNull_thenReturnZonedInEuropeMoscow() {
        LocalDateTime dt = LocalDateTime.of(2026, 2, 17, 10, 11, 12);

        ZonedDateTime zdt = mapper.toZoned(dt);

        assertThat(zdt).isNotNull();
        assertThat(zdt.toLocalDateTime()).isEqualTo(dt);
        assertThat(zdt.getZone()).isEqualTo(ZoneId.of("Europe/Moscow"));
    }
}

```
