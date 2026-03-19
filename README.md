```java

@SpringBootTest
class SecurityConfigTest {

    @Autowired
    private ApplicationContext applicationContext;

    @MockBean
    private UserExtractionFilter userExtractionFilter;

    @Test
    void shouldLoadSecurityConfig() {
        SecurityFilterChain securityFilterChain = applicationContext.getBean(SecurityFilterChain.class);

        assertThat(securityFilterChain).isNotNull();
    }
}
```
