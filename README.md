```java
@Component
@RequiredArgsConstructor
@ConditionalOnProperty(prefix = "app.kafka", name = "enabled", havingValue = "true")
public class TrackerKafkaProducer {

    @Value("${app.tracker.kafka.topic.scheme:tracker_scheme}")
    private String schemeTopic;

    @Value("${app.tracker.kafka.topic.history:tracker_history}")
    private String historyTopic;

    @Value("${app.tracker.kafka.topic.additional:tracker_additional}")
    private String additionalTopic;

    private final KafkaTemplate<String, SchemeCreateDto> schemeCreateTemplate;
    private final KafkaTemplate<String, HistoryNewDto> historyNewTemplate;
    private final KafkaTemplate<String, PlannedDateDto> plannedDateTemplate;

    public void sendScheme(SchemeCreateDto dto) {
        schemeCreateTemplate.send(schemeTopic, StsTrackerSchemeCodes.STATUS_STS, dto);

        log.info("Схема статусов отправлена в Kafka. topic={}", schemeTopic);
    }

    public void sendHistory(HistoryNewDto dto) {
        String key = dto.getEntityUuid().toString();

        historyNewTemplate.send(historyTopic, key, dto);

        log.info("История статуса отправлена в Kafka. topic={}, entityUuid={}, status={}",
                historyTopic, dto.getEntityUuid(), dto.getStatus());
    }

    public void sendAdditional(PlannedDateDto dto) {
        String key = dto.getEntityUuid().toString();

        plannedDateTemplate.send(additionalTopic, key, dto);

        log.info("Planned date отправлена в Kafka. topic={}, entityUuid={}",
                additionalTopic, dto.getEntityUuid());
    }
}
```
