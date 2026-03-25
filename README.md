```java
@Slf4j
@Component
@RequiredArgsConstructor
public class TrackerSchemeKafkaProducer {

    @Value("${app.tracker.kafka.topic.scheme:tracker_scheme}")
    private String schemeTopic;

    private final KafkaTemplate<String, SchemeCreateDto> schemeCreateTemplate;

    public void send(SchemeCreateDto dto) {
        schemeCreateTemplate.send(schemeTopic, StsTrackerSchemeCodes.STATUS_STS, dto);

        log.info("Схема статусов отправлена в Kafka. topic={}", schemeTopic);
    }
}

@Slf4j
@Component
@RequiredArgsConstructor
public class TrackerHistoryKafkaProducer {

    @Value("${app.tracker.kafka.topic.history:tracker_history}")
    private String historyTopic;

    private final KafkaTemplate<String, HistoryNewDto> historyNewTemplate;

    public void send(HistoryNewDto dto) {
        String key = dto.getEntityUuid().toString();

        historyNewTemplate.send(historyTopic, key, dto);

        log.info("История статуса отправлена в Kafka. topic={}, entityUuid={}, status={}",
                historyTopic, dto.getEntityUuid(), dto.getStatus());
    }
}

@Slf4j
@Component
@RequiredArgsConstructor
public class TrackerSchemeRegistrationService {

    private static final UUID SYSTEM_USER_UUID = UUID.fromString("00000000-0000-0000-0000-000000000000");
    private static final String SYSTEM_USER = "control-vehicle-app";

    @Value("classpath:tracker/status-sts.yml")
    private Resource schemeResource;

    private final TrackerSchemeKafkaProducer trackerSchemeKafkaProducer;

    @EventListener(ApplicationReadyEvent.class)
    public void publishScheme() {
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

@Slf4j
@Component
@RequiredArgsConstructor
public class TrackerAdditionalKafkaProducer {

    @Value("${app.tracker.kafka.topic.additional:tracker_additional}")
    private String additionalTopic;

    private final KafkaTemplate<String, PlannedDateDto> plannedDateTemplate;

    public void send(PlannedDateDto dto) {
        String key = dto.getEntityUuid().toString();

        plannedDateTemplate.send(additionalTopic, key, dto);

        log.info("Planned date отправлена в Kafka. topic={}, entityUuid={}",
                additionalTopic, dto.getEntityUuid());
    }
}
```
