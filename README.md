```java
@ExtendWith(MockitoExtension.class)
class SendTrackerHistoryOutboxMessageActionTest {

    @Mock
    private TrackerKafkaProducer trackerKafkaProducer;

    @Mock
    private OutboxMessagePersistenceService outboxMessagePersistenceService;

    private SendTrackerHistoryOutboxMessageAction action;
    private ObjectMapper objectMapper;

    @BeforeEach
    void setUp() {
        objectMapper = new ObjectMapper().findAndRegisterModules();

        action = new SendTrackerHistoryOutboxMessageAction(
                trackerKafkaProducer,
                outboxMessagePersistenceService,
                objectMapper
        );

        ReflectionTestUtils.setField(action, "initialRetryDelay", 10);
        ReflectionTestUtils.setField(action, "maximumRetryDelay", 14400);
        ReflectionTestUtils.setField(action, "maximumOfRetries", 10);
    }

    @Test
    void execute_shouldReadPayloadCallTrackerKafkaProducerAndMarkMessageDone() throws Exception {
        UUID messageId = UUID.randomUUID();
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

        OutboxMessage message = new OutboxMessage(
                OutboxMessageEventType.SEND_TRACKER_HISTORY,
                createdBy,
                entityUuid.toString(),
                objectMapper.writeValueAsString(payload)
        );

        ReflectionTestUtils.setField(message, "uuid", messageId);
        ReflectionTestUtils.setField(message, "status", OutboxMessageStatus.IN_PROGRESS);

        when(outboxMessagePersistenceService.findUnpublishedMessageByIdWithErrors(messageId))
                .thenReturn(Optional.of(message));

        when(outboxMessagePersistenceService.save(any(OutboxMessage.class)))
                .thenAnswer(invocation -> invocation.getArgument(0));

        action.execute(message.getUuid());

        ArgumentCaptor<HistoryNewDto> payloadCaptor =
                ArgumentCaptor.forClass(HistoryNewDto.class);

        verify(trackerKafkaProducer).sendHistory(payloadCaptor.capture());

        HistoryNewDto sentPayload = payloadCaptor.getValue();

        assertEquals(StsTrackerSchemeCodes.STATUS_STS, sentPayload.getCode());
        assertEquals(entityUuid, sentPayload.getEntityUuid());
        assertEquals(StsAction.CREATE_STS.getId(), sentPayload.getOperation());
        assertEquals(StsStatus.DRAFT.name(), sentPayload.getStatus());
        assertNull(sentPayload.getComment());
        assertEquals(createdBy, sentPayload.getCreatedBy());
        assertEquals(createdBy.toString(), sentPayload.getExtCreatedBy());
        assertEquals(createdBy.toString(), sentPayload.getUserName());
        assertNull(sentPayload.getUserPosition());

        verify(outboxMessagePersistenceService).findUnpublishedMessageByIdWithErrors(messageId);
        verify(outboxMessagePersistenceService).save(argThat(saved ->
                saved.getStatus() == OutboxMessageStatus.DONE
                        && saved.getPublishedAt() != null
        ));
    }

    @Test
    void execute_shouldDoNothingWhenMessageNotFound() {
        UUID messageId = UUID.randomUUID();

        when(outboxMessagePersistenceService.findUnpublishedMessageByIdWithErrors(messageId))
                .thenReturn(Optional.empty());

        action.execute(messageId);

        verify(outboxMessagePersistenceService).findUnpublishedMessageByIdWithErrors(messageId);
        verifyNoInteractions(trackerKafkaProducer);
        verify(outboxMessagePersistenceService, never()).save(any());
    }
}
```
