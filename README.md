```java
@ExtendWith(MockitoExtension.class)
class CreateEventOutboxMessageActionTest {

    @Mock
    private EventsHistoryClient eventsHistoryClient;

    @Mock
    private OutboxMessagePersistenceService outboxMessagePersistenceService;

    private CreateEventOutboxMessageAction action;
    private ObjectMapper objectMapper;

    @BeforeEach
    void setUp() {
        objectMapper = new ObjectMapper().findAndRegisterModules();

        action = new CreateEventOutboxMessageAction(
                eventsHistoryClient,
                outboxMessagePersistenceService,
                objectMapper
        );

        ReflectionTestUtils.setField(action, "initialRetryDelay", 10);
        ReflectionTestUtils.setField(action, "maximumRetryDelay", 14400);
        ReflectionTestUtils.setField(action, "maximumOfRetries", 10);
    }

    @Test
    void execute_shouldReadPayloadCallEventsHistoryClientAndMarkMessageDone() throws Exception {
        UUID messageId = UUID.randomUUID();
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

        OutboxMessage message = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
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

        ArgumentCaptor<EventCreateDto> eventCaptor =
                ArgumentCaptor.forClass(EventCreateDto.class);

        verify(eventsHistoryClient).createEvent(eventCaptor.capture());

        EventCreateDto sentEvent = eventCaptor.getValue();

        assertEquals(StsHistoryEventIds.SERVICE_ID, sentEvent.getServiceId());
        assertEquals(StsHistoryEventIds.STS_CREATE, sentEvent.getId());
        assertEquals(entityUuid, sentEvent.getEntityUuid());
        assertEquals(createdBy.toString(), sentEvent.getSubmittedBy());

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
        verifyNoInteractions(eventsHistoryClient);
        verify(outboxMessagePersistenceService, never()).save(any());
    }
}
```
