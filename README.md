```java
class JpaConfigTest {

    private static final String USER_PROFILE_UUID = "USER_PROFILE_UUID";

    private final JpaConfig jpaConfig = new JpaConfig();

    @Test
    void zonedDateTimeProvider_shouldReturnCurrentZonedDateTime() {
        DateTimeProvider provider = jpaConfig.zonedDateTimeProvider();

        Optional<TemporalAccessor> now = provider.getNow();

        assertThat(now).isPresent();
        assertThat(now.get()).isInstanceOf(ZonedDateTime.class);
    }

```
