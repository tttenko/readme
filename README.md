```java

class SecurityConfigTest {

    private final WebApplicationContextRunner contextRunner = new WebApplicationContextRunner()
            .withConfiguration(AutoConfigurations.of(
                    SecurityAutoConfiguration.class,
                    SecurityFilterAutoConfiguration.class
            ))
            .withUserConfiguration(SecurityConfig.class, TestBeans.class);

    @Test
    void shouldCreateSecurityFilterChain() {
        contextRunner.run(context ->
                assertThat(context).hasSingleBean(SecurityFilterChain.class)
        );
    }

    @TestConfiguration
    static class TestBeans {

        @Bean
        UserExtractionFilter userExtractionFilter() {
            return Mockito.mock(UserExtractionFilter.class);
        }
    }
}
```
