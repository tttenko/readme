```java
@SpringBootTest(
        properties = {
                "outbox.enabled=true",
                "app.kafka.enabled=true",
                "spring.task.scheduling.enabled=false",
                "app.db.schema=public",
                "events-history.client.url=http://localhost:9999"
        }
)
@Testcontainers
class OutboxFlowIntegrationTest {

    @Container
    static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void registerProperties(DynamicPropertyRegistry registry) {
        registry.add("db.url", POSTGRES::getJdbcUrl);
        registry.add("db.username", POSTGRES::getUsername);
        registry.add("db.password", POSTGRES::getPassword);
    }

    @Autowired
    private OutboxMessageService outboxMessageService;

    @Autowired
    private OutboxMessageProcessor outboxMessageProcessor;

    @Autowired
    private OutboxMessageRepository outboxMessageRepository;

    @MockBean
    private EventsHistoryClient eventsHistoryClient;

    @MockBean
    private TrackerKafkaProducer trackerKafkaProducer;

    @AfterEach
    void tearDown() {
        outboxMessageRepository.deleteAll();
    }

    @Test
    void shouldProcessCreateEventAndMarkOutboxMessageDone() {
        UUID entityUuid = UUID.randomUUID();
        UUID createdBy = UUID.randomUUID();

        EventCreateDto payload = EventCreateDto.builder()
                .serviceId(StsHistoryEventIds.SERVICE_ID)
                .id(StsHistoryEventIds.STS_CREATE)
                .entityUuid(entityUuid)
                .submittedAt(ZonedDateTime.now())
                .submittedBy(createdBy.toString())
                .session("NO-SESSION")
                .userName("Иван Иванов")
                .userNode("NO-USERNODE")
                .parameters(List.of(
                        EventParameterCreateDto.builder()
                                .id("vehicle_num")
                                .value("A123AA77")
                                .build(),
                        EventParameterCreateDto.builder()
                                .id("vehicle_name")
                                .value("Камаз")
                                .build()
                ))
                .build();

        OutboxMessage message = outboxMessageService.addOutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                createdBy,
                entityUuid.toString(),
                payload
        );

        OutboxMessage pendingMessage = outboxMessageRepository.findById(message.getUuid()).orElseThrow();

        assertThat(pendingMessage.getStatus()).isEqualTo(OutboxMessageStatus.PENDING);
        assertThat(pendingMessage.getPublishedAt()).isNull();

        outboxMessageProcessor.processOutboxMessage(message.getUuid());

        ArgumentCaptor<EventCreateDto> captor = ArgumentCaptor.forClass(EventCreateDto.class);
        verify(eventsHistoryClient).createEvent(captor.capture());
        verifyNoInteractions(trackerKafkaProducer);

        EventCreateDto sentPayload = captor.getValue();

        assertThat(sentPayload.getServiceId()).isEqualTo(StsHistoryEventIds.SERVICE_ID);
        assertThat(sentPayload.getId()).isEqualTo(StsHistoryEventIds.STS_CREATE);
        assertThat(sentPayload.getEntityUuid()).isEqualTo(entityUuid);
        assertThat(sentPayload.getSubmittedBy()).isEqualTo(createdBy.toString());

        OutboxMessage doneMessage = outboxMessageRepository.findById(message.getUuid()).orElseThrow();

        assertThat(doneMessage.getStatus()).isEqualTo(OutboxMessageStatus.DONE);
        assertThat(doneMessage.getPublishedAt()).isNotNull();
    }

    @Test
    void shouldProcessSendTrackerHistoryAndMarkOutboxMessageDone() {
        UUID entityUuid = UUID.randomUUID();
        UUID createdBy = UUID.randomUUID();

        HistoryNewDto payload = HistoryNewDto.builder()
                .code(StsTrackerSchemeCodes.STATUS_STS)
                .entityUuid(entityUuid)
                .operation(StsAction.CREATE_STS.getId())
                .status(StsStatus.DRAFT.name())
                .comment(null)
                .createdBy(createdBy)
                .extCreatedBy(createdBy.toString())
                .userName(createdBy.toString())
                .userPosition(null)
                .build();

        OutboxMessage message = outboxMessageService.addOutboxMessage(
                OutboxMessageEventType.SEND_TRACKER_HISTORY,
                createdBy,
                entityUuid.toString(),
                payload
        );

        OutboxMessage pendingMessage = outboxMessageRepository.findById(message.getUuid()).orElseThrow();

        assertThat(pendingMessage.getStatus()).isEqualTo(OutboxMessageStatus.PENDING);
        assertThat(pendingMessage.getPublishedAt()).isNull();

        outboxMessageProcessor.processOutboxMessage(message.getUuid());

        ArgumentCaptor<HistoryNewDto> captor = ArgumentCaptor.forClass(HistoryNewDto.class);
        verify(trackerKafkaProducer).sendHistory(captor.capture());
        verifyNoInteractions(eventsHistoryClient);

        HistoryNewDto sentPayload = captor.getValue();

        assertThat(sentPayload.getCode()).isEqualTo(StsTrackerSchemeCodes.STATUS_STS);
        assertThat(sentPayload.getEntityUuid()).isEqualTo(entityUuid);
        assertThat(sentPayload.getOperation()).isEqualTo(StsAction.CREATE_STS.getId());
        assertThat(sentPayload.getStatus()).isEqualTo(StsStatus.DRAFT.name());
        assertThat(sentPayload.getCreatedBy()).isEqualTo(createdBy);
        assertThat(sentPayload.getExtCreatedBy()).isEqualTo(createdBy.toString());
        assertThat(sentPayload.getUserName()).isEqualTo(createdBy.toString());

        OutboxMessage doneMessage = outboxMessageRepository.findById(message.getUuid()).orElseThrow();

        assertThat(doneMessage.getStatus()).isEqualTo(OutboxMessageStatus.DONE);
        assertThat(doneMessage.getPublishedAt()).isNotNull();
    }
}
```
