```java
@Configuration
@RequiredArgsConstructor
public class StsTrackerSchemeConfig {

    private final PathSchemeFactory pathSchemeFactory;

    @Value("${tracker.scheme.path}")
    private String schemePath;

    @Bean
    public Scheme stsTrackerScheme() {
        return pathSchemeFactory.createInstance(schemePath);
    }
}
```
