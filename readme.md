```java
@Slf4j
@Configuration
@Profile("!local")       // все профили, кроме local
public class ApplicationConfig {

    /**
     * Обычный WebClient без сертификата (бывшая else-ветка).
     */
    @Bean
    public WebClient webClient(ConnectionProperties props) {
        return WebClient.builder()
                .codecs(configure -> configure
                        .defaultCodecs()
                        .maxInMemorySize(getBufferSize(props)))
                .build();
    }
}
```
