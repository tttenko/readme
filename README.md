```java
@Slf4j
@Component
@RequiredArgsConstructor
public class TrackerSchemeRegistrationService {

    private static final UUID SYSTEM_USER_UUID = UUID.fromString("00000000-0000-0000-0000-000000000000");
    private static final String SYSTEM_USER = "control-vehicle-app";

    @Value("${app.tracker.kafka.scheme.publish-on-startup:true}")
    private boolean publishSchemeOnStartup;

    @Value("classpath:tracker/status-sts.yml")
    private Resource schemeResource;

    private final TrackerSchemeKafkaProducer trackerSchemeKafkaProducer;

    @EventListener(ApplicationReadyEvent.class)
    public void publishScheme() {
        if (!publishSchemeOnStartup) {
            log.info("Публикация схемы при старте отключена");
            return;
        }

        try {
            String yaml = new String(schemeResource.getInputStream().readAllBytes(), StandardCharsets.UTF_8);

            SchemeCreateDto dto = SchemeCreateDto.builder()
                    .data(yaml)
                    .createdBy(SYSTEM_USER_UUID)
                    .extCreatedBy(SYSTEM_USER)
                    .build();

            trackerSchemeKafkaProducer.send(dto);
        } catch (IOException ex) {
            throw new IllegalStateException("Не удалось прочитать файл статусной схемы", ex);
        }
    }
}
```
