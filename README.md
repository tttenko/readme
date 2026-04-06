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
        // given
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

        // when
        stsTrackerHistoryOutboxService.sendCreatedStatus(entity);

        // then
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
                () -> assertEquals(outboxUuid, publishedEvent.getUuid())
        );
    }

    @Test
    void sendCreatedStatus_shouldPublishOutboxMessageEventWithReturnedMessageUuid() {
        // given
        UUID entityUuid = UUID.randomUUID();
        UUID createdBy = UUID.randomUUID();
        UUID outboxUuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setCreatedBy(createdBy);
        entity.setStatusId(StsStatus.DRAFT);

        OutboxMessage outboxMessage = mock(OutboxMessage.class);
        when(outboxMessage.getUuid()).thenReturn(outboxUuid);

        when(outboxMessageService.addOutboxMessage(any(), any(), any(), any()))
                .thenReturn(outboxMessage);

        // when
        stsTrackerHistoryOutboxService.sendCreatedStatus(entity);

        // then
        verify(applicationEventPublisher)
                .publishEvent(new OutboxMessageEvent(outboxUuid));
    }

    @Test
    void sendCreatedStatus_whenOutboxServiceThrows_thenPropagateExceptionAndDoNotPublishEvent() {
        // given
        UUID entityUuid = UUID.randomUUID();
        UUID createdBy = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setCreatedBy(createdBy);
        entity.setStatusId(StsStatus.DRAFT);

        OutboxMessageActionException exception =
                new OutboxMessageActionException(
                        "Unable to serialize payload for outbox message",
                        new RuntimeException()
                );

        doThrow(exception).when(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.SEND_TRACKER_HISTORY),
                eq(createdBy),
                eq(entityUuid.toString()),
                any(HistoryNewDto.class)
        );

        // when / then
        assertThatThrownBy(() -> stsTrackerHistoryOutboxService.sendCreatedStatus(entity))
                .isInstanceOf(OutboxMessageActionException.class)
                .hasMessage("Unable to serialize payload for outbox message");

        verify(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.SEND_TRACKER_HISTORY),
                eq(createdBy),
                eq(entityUuid.toString()),
                any(HistoryNewDto.class)
        );

        verifyNoInteractions(applicationEventPublisher);
        verifyNoMoreInteractions(outboxMessageService);
    }

    @Test
    void sendToApproveStatus_shouldBuildDtoSaveOutboxMessageAndPublishEvent() {
        // given
        UUID entityUuid = UUID.randomUUID();
        UUID updatedBy = UUID.randomUUID();
        UUID outboxUuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setUpdatedBy(updatedBy);
        entity.setStatusId(StsStatus.TO_APPROVE_IN);

        OutboxMessage outboxMessage = mock(OutboxMessage.class);
        when(outboxMessage.getUuid()).thenReturn(outboxUuid);

        when(outboxMessageService.addOutboxMessage(
                eq(OutboxMessageEventType.SEND_TRACKER_HISTORY),
                eq(updatedBy),
                eq(entityUuid.toString()),
                historyCaptor.capture()
        )).thenReturn(outboxMessage);

        // when
        stsTrackerHistoryOutboxService.sendToApproveStatus(entity);

        // then
        verify(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.SEND_TRACKER_HISTORY),
                eq(updatedBy),
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
                () -> assertEquals(StsAction.TO_APPROVE.getId(), dto.getOperation()),
                () -> assertEquals(StsStatus.TO_APPROVE_IN.name(), dto.getStatus()),
                () -> assertNull(dto.getComment()),
                () -> assertEquals(updatedBy, dto.getCreatedBy()),
                () -> assertEquals(updatedBy.toString(), dto.getExtCreatedBy()),
                () -> assertEquals(updatedBy.toString(), dto.getUserName()),
                () -> assertNull(dto.getUserPosition()),
                () -> assertEquals(outboxUuid, publishedEvent.getUuid())
        );
    }

    @Test
    void sendToApproveStatus_shouldPublishOutboxMessageEventWithReturnedMessageUuid() {
        // given
        UUID entityUuid = UUID.randomUUID();
        UUID updatedBy = UUID.randomUUID();
        UUID outboxUuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setUpdatedBy(updatedBy);
        entity.setStatusId(StsStatus.TO_APPROVE_OUT);

        OutboxMessage outboxMessage = mock(OutboxMessage.class);
        when(outboxMessage.getUuid()).thenReturn(outboxUuid);

        when(outboxMessageService.addOutboxMessage(any(), any(), any(), any()))
                .thenReturn(outboxMessage);

        // when
        stsTrackerHistoryOutboxService.sendToApproveStatus(entity);

        // then
        verify(applicationEventPublisher)
                .publishEvent(new OutboxMessageEvent(outboxUuid));
    }

    @Test
    void sendToApproveStatus_whenOutboxServiceThrows_thenPropagateExceptionAndDoNotPublishEvent() {
        // given
        UUID entityUuid = UUID.randomUUID();
        UUID updatedBy = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setUpdatedBy(updatedBy);
        entity.setStatusId(StsStatus.TO_APPROVE_OUT);

        OutboxMessageActionException exception =
                new OutboxMessageActionException(
                        "Unable to serialize payload for outbox message",
                        new RuntimeException()
                );

        doThrow(exception).when(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.SEND_TRACKER_HISTORY),
                eq(updatedBy),
                eq(entityUuid.toString()),
                any(HistoryNewDto.class)
        );

        // when / then
        assertThatThrownBy(() -> stsTrackerHistoryOutboxService.sendToApproveStatus(entity))
                .isInstanceOf(OutboxMessageActionException.class)
                .hasMessage("Unable to serialize payload for outbox message");

        verify(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.SEND_TRACKER_HISTORY),
                eq(updatedBy),
                eq(entityUuid.toString()),
                any(HistoryNewDto.class)
        );

        verifyNoInteractions(applicationEventPublisher);
        verifyNoMoreInteractions(outboxMessageService);
    }
}
```
