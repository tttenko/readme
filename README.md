```java

@SpringBootTest(classes = SecurityConfigTest.TestApp.class)
class SecurityConfigTest {

    @Autowired
    private SecurityFilterChain securityFilterChain;

    @MockBean
    private UserExtractionFilter userExtractionFilter;

    @Test
    void shouldLoadSecurityConfig() {
        assertThat(securityFilterChain).isNotNull();
    }

    @SpringBootConfiguration
    @EnableAutoConfiguration
    @Import(SecurityConfig.class)
    static class TestApp {
    }
}
```
