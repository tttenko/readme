```java
@ExtendWith(MockitoExtension.class)
class StsTrackerHistoryOutboxServiceTest {

    @Mock
    private OutboxMessageService outboxMessageService;

    @Mock
    private ApplicationEventPublisher applicationEventPublisher;

    @InjectMocks
    private StsTrackerHistoryOutboxService stsTrackerHistoryOutboxService;

    @Captor
    private ArgumentCaptor<HistoryNewDto> historyCaptor;

    @Captor
    private ArgumentCaptor<OutboxMessageEvent> outboxEventCaptor;

    @Test
    void sendCreatedStatus_shouldBuildDtoSaveOutboxMessageAndPublishEvent() {
        UUID entityUuid = UUID.randomUUID();
        UUID createdBy = UUID.randomUUID();
        UUID outboxUuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setCreatedBy(createdBy);
        entity.setStatusId(StsStatus.DRAFT);

        OutboxMessage outboxMessage = mock(OutboxMessage.class);
        when(outboxMessage.getUuid()).thenReturn(outboxUuid);

        when(outboxMessageService.addOutboxMessage(
                eq(OutboxMessageEventType.SEND_TRACKER_HISTORY),
                eq(createdBy),
                eq(entityUuid.toString()),
                historyCaptor.capture()
        )).thenReturn(outboxMessage);

        stsTrackerHistoryOutboxService.sendCreatedStatus(entity);

        verify(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.SEND_TRACKER_HISTORY),
                eq(createdBy),
                eq(entityUuid.toString()),
                any(HistoryNewDto.class)
        );

        verify(applicationEventPublisher).publishEvent(outboxEventCaptor.capture());
        verifyNoMoreInteractions(outboxMessageService, applicationEventPublisher);

        HistoryNewDto dto = historyCaptor.getValue();
        OutboxMessageEvent publishedEvent = outboxEventCaptor.getValue();

        assertAll(
                () -> assertEquals(StsTrackerSchemeCodes.STATUS_STS, dto.getCode()),
                () -> assertEquals(entityUuid, dto.getEntityUuid()),
                () -> assertEquals(StsAction.CREATE_STS.getId(), dto.getOperation()),
                () -> assertEquals(StsStatus.DRAFT.name(), dto.getStatus()),
                () -> assertNull(dto.getComment()),
                () -> assertEquals(createdBy, dto.getCreatedBy()),
                () -> assertEquals(createdBy.toString(), dto.getExtCreatedBy()),
                () -> assertEquals(createdBy.toString(), dto.getUserName()),
                () -> assertNull(dto.getUserPosition()),
                () -> assertEquals(outboxUuid, publishedEvent.uuid())
        );
    }


    @ExtendWith(MockitoExtension.class)
class StsTrackerHistoryOutboxServiceTest {

    @Mock
    private OutboxMessageService outboxMessageService;

    @Mock
    private ApplicationEventPublisher applicationEventPublisher;

    @InjectMocks
    private StsTrackerHistoryOutboxService stsTrackerHistoryOutboxService;

    @Captor
    private ArgumentCaptor<HistoryNewDto> historyCaptor;

    @Captor
    private ArgumentCaptor<OutboxMessageEvent> outboxEventCaptor;

    @Test
    void sendCreatedStatus_shouldBuildDtoSaveOutboxMessageAndPublishEvent() {
        UUID entityUuid = UUID.randomUUID();
        UUID createdBy = UUID.randomUUID();
        UUID outboxUuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setCreatedBy(createdBy);
        entity.setStatusId(StsStatus.DRAFT);

        OutboxMessage outboxMessage = mock(OutboxMessage.class);
        when(outboxMessage.getUuid()).thenReturn(outboxUuid);

        when(outboxMessageService.addOutboxMessage(
                eq(OutboxMessageEventType.SEND_TRACKER_HISTORY),
                eq(createdBy),
                eq(entityUuid.toString()),
                historyCaptor.capture()
        )).thenReturn(outboxMessage);

        stsTrackerHistoryOutboxService.sendCreatedStatus(entity);

        verify(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.SEND_TRACKER_HISTORY),
                eq(createdBy),
                eq(entityUuid.toString()),
                any(HistoryNewDto.class)
        );

        verify(applicationEventPublisher).publishEvent(outboxEventCaptor.capture());
        verifyNoMoreInteractions(outboxMessageService, applicationEventPublisher);

        HistoryNewDto dto = historyCaptor.getValue();
        OutboxMessageEvent publishedEvent = outboxEventCaptor.getValue();

        assertAll(
                () -> assertEquals(StsTrackerSchemeCodes.STATUS_STS, dto.getCode()),
                () -> assertEquals(entityUuid, dto.getEntityUuid()),
                () -> assertEquals(StsAction.CREATE_STS.getId(), dto.getOperation()),
                () -> assertEquals(StsStatus.DRAFT.name(), dto.getStatus()),
                () -> assertNull(dto.getComment()),
                () -> assertEquals(createdBy, dto.getCreatedBy()),
                () -> assertEquals(createdBy.toString(), dto.getExtCreatedBy()),
                () -> assertEquals(createdBy.toString(), dto.getUserName()),
                () -> assertNull(dto.getUserPosition()),
                () -> assertEquals(outboxUuid, publishedEvent.uuid())
        );
    }
}
}
```
