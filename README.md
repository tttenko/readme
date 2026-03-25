```java
@Getter
@RequiredArgsConstructor
public enum StsStatus {

    DRAFT("DRAFT", "Черновик"),
    TO_APPROVE_IN("TO_APPROVE_IN", "На согласовании включения"),
    TO_APPROVE_OUT("TO_APPROVE_OUT", "На согласовании удаления"),
    APPROVE("APPROVE", "Включен в договор"),
    DELETE("DELETE", "Удален");

    private final String id;
    private final String title;

    public static StsStatus byId(String id) {
        return Arrays.stream(values())
                .filter(status -> Objects.equals(status.id, id))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("неизвестный статус СТС: " + id));
    }

    public static String titleById(String id) {
        return byId(id).getTitle();
    }
}

@Getter
@RequiredArgsConstructor
public enum StsStatus {

    DRAFT("DRAFT", "Черновик"),
    TO_APPROVE_IN("TO_APPROVE_IN", "На согласовании включения"),
    TO_APPROVE_OUT("TO_APPROVE_OUT", "На согласовании удаления"),
    APPROVE("APPROVE", "Включен в договор"),
    DELETE("DELETE", "Удален");

    private final String id;
    private final String title;

    public static StsStatus byId(String id) {
        return Arrays.stream(values())
                .filter(status -> Objects.equals(status.id, id))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("неизвестный статус СТС: " + id));
    }

    public static String titleById(String id) {
        return byId(id).getTitle();
    }
}

@Getter
@RequiredArgsConstructor
public enum StsAction {

    TO_APPROVE("toApprove", "Индикатор отправки на согласование включения СТС"),
    DELETE("delete", "Индикатор удаления СТС"),
    CREATE_STS("createSts", "Индикатор создания СТС"),
    APPROVE("approve", "Индикатор согласования СТС"),
    REJECT("reject", "Индикатор отклонения СТС"),
    UPDATE("update", "Индикатор изменения СТС");

    private final String id;
    private final String description;
}

@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class StsTrackerSchemeCodes {

    public static final String STATUS_STS = "status_sts";
}

public record StsCreatedTrackerHistoryEvent(
        UUID entityUuid,
        String statusId,
        ZonedDateTime changedAt,
        UUID createdBy
) {
    public static StsCreatedTrackerHistoryEvent from(StsDataEntity entity) {
        return new StsCreatedTrackerHistoryEvent(
                entity.getUuid(),
                entity.getStatusId(),
                entity.getCreatedAt(),
                entity.getCreatedBy()
        );
    }
}

@Getter
@Setter
@ConfigurationProperties(prefix = "app.tracker.kafka")
public class TrackerKafkaProperties {

    private Topic topic = new Topic();
    private Send send = new Send();
    private Scheme scheme = new Scheme();

    @Getter
    @Setter
    public static class Topic {
        private String scheme;
        private String history;
        private String additional;
    }

    @Getter
    @Setter
    public static class Send {
        private boolean active;
    }

    @Getter
    @Setter
    public static class Scheme {
        private boolean publishOnStartup;
    }
}

@Configuration
@EnableConfigurationProperties(TrackerKafkaProperties.class)
public class TrackerPropertiesConfig {
}

@Configuration
public class KafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public Map<String, Object> producerConfigs() {
        Map<String, Object> props = new HashMap<>();

        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        return props;
    }

    @Bean
    public ProducerFactory<String, SchemeCreateDto> schemeCreateProducerFactory() {
        return new DefaultKafkaProducerFactory<>(producerConfigs());
    }

    @Bean
    public ProducerFactory<String, HistoryNewDto> historyNewProducerFactory() {
        return new DefaultKafkaProducerFactory<>(producerConfigs());
    }

    @Bean
    public ProducerFactory<String, PlannedDateDto> plannedDateProducerFactory() {
        return new DefaultKafkaProducerFactory<>(producerConfigs());
    }

    @Bean
    public KafkaTemplate<String, SchemeCreateDto> schemeCreateTemplate() {
        return new KafkaTemplate<>(schemeCreateProducerFactory());
    }

    @Bean
    public KafkaTemplate<String, HistoryNewDto> historyNewTemplate() {
        return new KafkaTemplate<>(historyNewProducerFactory());
    }

    @Bean
    public KafkaTemplate<String, PlannedDateDto> plannedDateTemplate() {
        return new KafkaTemplate<>(plannedDateProducerFactory());
    }
}

@Slf4j
@Component
@RequiredArgsConstructor
public class TrackerSchemeKafkaProducer {

    private final KafkaTemplate<String, SchemeCreateDto> schemeCreateTemplate;
    private final TrackerKafkaProperties trackerKafkaProperties;

    public void send(SchemeCreateDto dto) {
        if (!trackerKafkaProperties.getSend().isActive()) {
            log.info("Kafka отправка схемы отключена");
            return;
        }

        String topic = trackerKafkaProperties.getTopic().getScheme();
        schemeCreateTemplate.send(topic, StsTrackerSchemeCodes.STATUS_STS, dto);

        log.info("Схема статусов отправлена в Kafka. topic={}", topic);
    }
}

Slf4j
@Component
@RequiredArgsConstructor
public class TrackerHistoryKafkaProducer {

    private final KafkaTemplate<String, HistoryNewDto> historyNewTemplate;
    private final TrackerKafkaProperties trackerKafkaProperties;

    public void send(HistoryNewDto dto) {
        if (!trackerKafkaProperties.getSend().isActive()) {
            log.info("Kafka отправка истории статусов отключена");
            return;
        }

        String topic = trackerKafkaProperties.getTopic().getHistory();
        String key = dto.getEntityUuid().toString();

        historyNewTemplate.send(topic, key, dto);

        log.info("История статуса отправлена в Kafka. topic={}, entityUuid={}, status={}",
                topic, dto.getEntityUuid(), dto.getStatus());
    }
}

@Slf4j
@Component
@RequiredArgsConstructor
public class TrackerAdditionalKafkaProducer {

    private final KafkaTemplate<String, PlannedDateDto> plannedDateTemplate;
    private final TrackerKafkaProperties trackerKafkaProperties;

    public void send(PlannedDateDto dto) {
        if (!trackerKafkaProperties.getSend().isActive()) {
            log.info("Kafka отправка planned date отключена");
            return;
        }

        String topic = trackerKafkaProperties.getTopic().getAdditional();
        String key = dto.getEntityUuid().toString();

        plannedDateTemplate.send(topic, key, dto);

        log.info("Planned date отправлена в Kafka. topic={}, entityUuid={}",
                topic, dto.getEntityUuid());
    }
}

@Slf4j
@Component
@RequiredArgsConstructor
public class TrackerAdditionalKafkaProducer {

    private final KafkaTemplate<String, PlannedDateDto> plannedDateTemplate;
    private final TrackerKafkaProperties trackerKafkaProperties;

    public void send(PlannedDateDto dto) {
        if (!trackerKafkaProperties.getSend().isActive()) {
            log.info("Kafka отправка planned date отключена");
            return;
        }

        String topic = trackerKafkaProperties.getTopic().getAdditional();
        String key = dto.getEntityUuid().toString();

        plannedDateTemplate.send(topic, key, dto);

        log.info("Planned date отправлена в Kafka. topic={}, entityUuid={}",
                topic, dto.getEntityUuid());
    }
}

@Service
@RequiredArgsConstructor
public class StsTrackerHistoryService {

    private final TrackerHistoryKafkaProducer trackerHistoryKafkaProducer;

    public void sendCreatedStatus(StsCreatedTrackerHistoryEvent event) {
        HistoryNewDto dto = HistoryNewDto.builder()
                .code(StsTrackerSchemeCodes.STATUS_STS)
                .entityUuid(event.entityUuid())
                .operation(StsAction.CREATE_STS.getId())
                .status(event.statusId())
                .comment(null)
                .createdBy(event.createdBy())
                .extCreatedBy(event.createdBy().toString())
                .userName(event.createdBy().toString())
                .userPosition(null)
                .build();

        trackerHistoryKafkaProducer.send(dto);
    }
}

@Component
@RequiredArgsConstructor
public class StsTrackerHistoryListener {

    private final StsTrackerHistoryService stsTrackerHistoryService;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handle(StsCreatedTrackerHistoryEvent event) {
        stsTrackerHistoryService.sendCreatedStatus(event);
    }
}

@Configuration
public class StsTrackerValidatorsConfig {

    @Bean("toapprovein")
    public SwitchCheck toApproveInValidator() {
        return (from, to, ctx) -> true;
    }

    @Bean("approvein")
    public SwitchCheck approveInValidator() {
        return (from, to, ctx) -> true;
    }

    @Bean("rejectin")
    public SwitchCheck rejectInValidator() {
        return (from, to, ctx) -> true;
    }

    @Bean("toapproveout")
    public SwitchCheck toApproveOutValidator() {
        return (from, to, ctx) -> true;
    }

    @Bean("approveout")
    public SwitchCheck approveOutValidator() {
        return (from, to, ctx) -> true;
    }

    @Bean("rejectout")
    public SwitchCheck rejectOutValidator() {
        return (from, to, ctx) -> true;
    }
}

spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

tracker:
  scheme:
    paths:
      - classpath:tracker/status-sts.yml

app:
  tracker:
    kafka:
      topic:
        scheme: ${app.tracker.kafka.topic.scheme:tracker_scheme}
        history: ${app.tracker.kafka.topic.history:tracker_history}
        additional: ${app.tracker.kafka.topic.additional:tracker_additional}
      send:
        active: ${app.tracker.kafka.send.active:true}
      scheme:
        publish-on-startup: ${app.tracker.kafka.scheme.publish-on-startup:true}

code: status_sts

nodes:
  - id: DRAFT
    start: true
    text: Черновик
    edges:
      - target: TO_APPROVE_IN
        id: TO_APPROVE_IN
        validator: toapprovein
        base: true

  - id: TO_APPROVE_IN
    text: На согласовании включения
    edges:
      - target: APPROVE
        id: APPROVE_IN
        validator: approvein
        base: true
      - target: DRAFT
        id: REJECT_IN
        validator: rejectin

  - id: APPROVE
    text: Включен в договор
    edges:
      - target: TO_APPROVE_OUT
        id: TO_APPROVE_OUT
        validator: toapproveout
        base: true

  - id: TO_APPROVE_OUT
    text: На согласовании удаления
    edges:
      - target: DELETE
        id: APPROVE_OUT
        validator: approveout
        base: true
      - target: APPROVE
        id: REJECT_OUT
        validator: rejectout

  - id: DELETE
    text: Удален
    end: true



```
