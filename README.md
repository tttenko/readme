```java
@Service
@RequiredArgsConstructor
@ConditionalOnProperty(prefix = "app.kafka", name = "enabled", havingValue = "true")
public class StsTrackerHistoryOutboxService {

    private final OutboxMessageService outboxMessageService;
    private final ApplicationEventPublisher applicationEventPublisher;

    /**
     * Сохраняет событие отправки статуса СТС в tracker через outbox.
     */
    public void sendCreatedStatus(StsDataEntity entity) {
        HistoryNewDto payload = HistoryNewDto.builder()
                .code(StsTrackerSchemeCodes.STATUS_STS)
                .entityUuid(entity.getUuid())
                .operation(StsAction.CREATE_STS.getId())
                .status(entity.getStatusId().name())
                .comment(null)
                .createdBy(entity.getCreatedBy())
                .extCreatedBy(entity.getCreatedBy().toString())
                .userName(entity.getCreatedBy().toString())
                .userPosition(null)
                .build();

        OutboxMessage message = outboxMessageService.addOutboxMessage(
                OutboxMessageEventType.SEND_TRACKER_HISTORY,
                entity.getCreatedBy(),
                entity.getUuid().toString(),
                payload
        );

        applicationEventPublisher.publishEvent(new OutboxMessageEvent(message.getUuid()));
    }
}

@Slf4j
@Component
@ConditionalOnProperty(prefix = "app.kafka", name = "enabled", havingValue = "true")
public class SendTrackerHistoryOutboxMessageAction extends OutboxMessageAction {

    private final TrackerKafkaProducer trackerKafkaProducer;
    private final ObjectMapper objectMapper;

    public SendTrackerHistoryOutboxMessageAction(
            TrackerKafkaProducer trackerKafkaProducer,
            OutboxMessagePersistenceService outboxMessagePersistenceService,
            ObjectMapper objectMapper
    ) {
        super(outboxMessagePersistenceService);
        this.trackerKafkaProducer = trackerKafkaProducer;
        this.objectMapper = objectMapper;
    }

    @Override
    public OutboxMessageEventType getEventType() {
        return OutboxMessageEventType.SEND_TRACKER_HISTORY;
    }

    @Override
    protected void action(OutboxMessage message) throws JsonProcessingException {
        HistoryNewDto payload = objectMapper.readValue(message.getPayload(), HistoryNewDto.class);
        trackerKafkaProducer.sendHistory(payload);
    }
}

@Slf4j
@Component
@RequiredArgsConstructor
@ConditionalOnProperty(prefix = "app.kafka", name = "enabled", havingValue = "true")
public class TrackerKafkaProducer {

    @Value("${app.tracker.kafka.topic.history:tracker_history}")
    private String historyTopic;

    @Value("${app.tracker.kafka.topic.additional:tracker_additional}")
    private String additionalTopic;

    private final KafkaTemplate<String, HistoryNewDto> historyNewTemplate;
    private final KafkaTemplate<String, PlannedDateDto> plannedDateTemplate;

    public void sendHistory(HistoryNewDto dto) {
        String key = dto.getEntityUuid().toString();

        try {
            historyNewTemplate.send(historyTopic, key, dto).get();

            log.info("История статуса отправлена в Kafka. topic={}, entityUuid={}, status={}",
                    historyTopic, dto.getEntityUuid(), dto.getStatus());
        } catch (Exception ex) {
            throw new IllegalStateException(
                    "Не удалось отправить событие tracker history в Kafka. entityUuid=" + dto.getEntityUuid(),
                    ex
            );
        }
    }

    public void sendAdditional(PlannedDateDto dto) {
        String key = dto.getEntityUuid().toString();

        try {
            plannedDateTemplate.send(additionalTopic, key, dto).get();

            log.info("Planned date отправлена в Kafka. topic={}, entityUuid={}",
                    additionalTopic, dto.getEntityUuid());
        } catch (Exception ex) {
            throw new IllegalStateException(
                    "Не удалось отправить additional сообщение в Kafka. entityUuid=" + dto.getEntityUuid(),
                    ex
            );
        }
    }
}
```
