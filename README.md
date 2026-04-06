```java
@ExtendWith(MockitoExtension.class)
class StsEventsHistoryServiceTest {

    private static final UUID AUTHOR_UUID = UUID.randomUUID();
    private static final String FULL_NAME = "Сенышин Осман Людовикович";

    @Mock
    private OutboxMessageService outboxMessageService;

    @Mock
    private ApplicationEventPublisher applicationEventPublisher;

    @InjectMocks
    private StsEventsHistoryOutboxService stsEventsHistoryService;

    @AfterEach
    void clearSecurityContext() {
        SecurityContextHolder.clearContext();
    }

    @Test
    void givenValidEntity_whenSendCreateEvent_thenSaveMessageToOutboxAndPublishEvent() {
        // given
        UUID entityUuid = UUID.randomUUID();
        UUID outboxMessageUuid = UUID.randomUUID();
        ZonedDateTime createdAt = ZonedDateTime.now();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setTbCode("1234");
        entity.setVehicleNumber("A123AA777");
        entity.setVehicleBrand("КАМАЗ");
        entity.setComment("Тестовая запись");
        entity.setStatusId(StsStatus.DRAFT);
        entity.setCreatedBy(AUTHOR_UUID);
        entity.setUpdatedBy(UUID.randomUUID());
        entity.setCreatedAt(createdAt);

        OutboxMessage outboxMessage = mock(OutboxMessage.class);
        when(outboxMessage.getUuid()).thenReturn(outboxMessageUuid);

        when(outboxMessageService.addOutboxMessage(
                eq(OutboxMessageEventType.CREATE_EVENT),
                eq(AUTHOR_UUID),
                eq(entityUuid.toString()),
                any(EventCreateDto.class)
        )).thenReturn(outboxMessage);

        mockSecurityContext(FULL_NAME);

        ArgumentCaptor<EventCreateDto> payloadCaptor = ArgumentCaptor.forClass(EventCreateDto.class);
        ArgumentCaptor<OutboxMessageEvent> eventCaptor = ArgumentCaptor.forClass(OutboxMessageEvent.class);

        // when
        stsEventsHistoryService.sendCreateEvent(entity);

        // then
        verify(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.CREATE_EVENT),
                eq(AUTHOR_UUID),
                eq(entityUuid.toString()),
                payloadCaptor.capture()
        );

        verify(applicationEventPublisher).publishEvent(eventCaptor.capture());
        verifyNoMoreInteractions(outboxMessageService, applicationEventPublisher);

        EventCreateDto payload = payloadCaptor.getValue();

        assertThat(payload.getServiceId()).isEqualTo(StsHistoryEventIds.SERVICE_ID);
        assertThat(payload.getId()).isEqualTo(StsHistoryEventIds.STS_CREATE);
        assertThat(payload.getEntityUuid()).isEqualTo(entityUuid);
        assertThat(payload.getSubmittedAt()).isEqualTo(createdAt);
        assertThat(payload.getSubmittedBy()).isEqualTo(AUTHOR_UUID.toString());
        assertThat(payload.getUserName()).isEqualTo(FULL_NAME);
        assertThat(payload.getSession()).isEqualTo("NO-SESSION");
        assertThat(payload.getUserNode()).isEqualTo("NO-USERNODE");

        Map<String, String> params = payload.getParameters().stream()
                .collect(Collectors.toMap(
                        EventParameterCreateDto::getId,
                        EventParameterCreateDto::getValue
                ));

        assertThat(params.get("vehicle_num")).isEqualTo("A123AA777");
        assertThat(params.get("vehicle_name")).isEqualTo("КАМАЗ");

        OutboxMessageEvent event = eventCaptor.getValue();
        assertThat(event.getUuid()).isEqualTo(outboxMessageUuid);
    }

    @Test
    void givenOutboxServiceThrows_whenSendCreateEvent_thenPropagateExceptionAndDoNotPublishEvent() {
        // given
        UUID entityUuid = UUID.randomUUID();
        ZonedDateTime createdAt = ZonedDateTime.now();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setVehicleNumber("A123AA777");
        entity.setVehicleBrand("КАМАЗ");
        entity.setStatusId(StsStatus.DRAFT);
        entity.setCreatedBy(AUTHOR_UUID);
        entity.setUpdatedBy(UUID.randomUUID());
        entity.setCreatedAt(createdAt);

        mockSecurityContext(FULL_NAME);

        OutboxMessageActionException exception =
                new OutboxMessageActionException(
                        "Unable to serialize payload for outbox message",
                        new RuntimeException()
                );

        doThrow(exception).when(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.CREATE_EVENT),
                eq(AUTHOR_UUID),
                eq(entityUuid.toString()),
                any(EventCreateDto.class)
        );

        // when / then
        assertThatThrownBy(() -> stsEventsHistoryService.sendCreateEvent(entity))
                .isInstanceOf(OutboxMessageActionException.class)
                .hasMessage("Unable to serialize payload for outbox message");

        verify(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.CREATE_EVENT),
                eq(AUTHOR_UUID),
                eq(entityUuid.toString()),
                any(EventCreateDto.class)
        );

        verifyNoInteractions(applicationEventPublisher);
        verifyNoMoreInteractions(outboxMessageService);
    }

    @Test
    void givenValidEntityAndOldStatus_whenSendToApproveEvent_thenSaveMessageToOutboxAndPublishEvent() {
        // given
        UUID entityUuid = UUID.randomUUID();
        UUID outboxMessageUuid = UUID.randomUUID();
        ZonedDateTime updatedAt = ZonedDateTime.now();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setTbCode("1234");
        entity.setVehicleNumber("A123AA777");
        entity.setVehicleBrand("КАМАЗ");
        entity.setStatusId(StsStatus.TO_APPROVE_IN);
        entity.setUpdatedBy(AUTHOR_UUID);
        entity.setUpdatedAt(updatedAt);

        OutboxMessage outboxMessage = mock(OutboxMessage.class);
        when(outboxMessage.getUuid()).thenReturn(outboxMessageUuid);

        when(outboxMessageService.addOutboxMessage(
                eq(OutboxMessageEventType.CREATE_EVENT),
                eq(AUTHOR_UUID),
                eq(entityUuid.toString()),
                any(EventCreateDto.class)
        )).thenReturn(outboxMessage);

        mockSecurityContext(FULL_NAME);

        ArgumentCaptor<EventCreateDto> payloadCaptor = ArgumentCaptor.forClass(EventCreateDto.class);
        ArgumentCaptor<OutboxMessageEvent> eventCaptor = ArgumentCaptor.forClass(OutboxMessageEvent.class);

        // when
        stsEventsHistoryService.sendToApproveEvent(entity, StsStatus.DRAFT);

        // then
        verify(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.CREATE_EVENT),
                eq(AUTHOR_UUID),
                eq(entityUuid.toString()),
                payloadCaptor.capture()
        );

        verify(applicationEventPublisher).publishEvent(eventCaptor.capture());
        verifyNoMoreInteractions(outboxMessageService, applicationEventPublisher);

        EventCreateDto payload = payloadCaptor.getValue();

        assertThat(payload.getServiceId()).isEqualTo(StsHistoryEventIds.SERVICE_ID);
        assertThat(payload.getId()).isEqualTo(StsHistoryEventIds.STS_UPDATE);
        assertThat(payload.getEntityUuid()).isEqualTo(entityUuid);
        assertThat(payload.getSubmittedAt()).isEqualTo(updatedAt);
        assertThat(payload.getSubmittedBy()).isEqualTo(AUTHOR_UUID.toString());
        assertThat(payload.getUserName()).isEqualTo(FULL_NAME);
        assertThat(payload.getSession()).isEqualTo("NO-SESSION");
        assertThat(payload.getUserNode()).isEqualTo("NO-USERNODE");

        Map<String, String> params = payload.getParameters().stream()
                .collect(Collectors.toMap(
                        EventParameterCreateDto::getId,
                        EventParameterCreateDto::getValue
                ));

        assertThat(params.get("vehicle_num")).isEqualTo("A123AA777");
        assertThat(params.get("vehicle_name")).isEqualTo("КАМАЗ");

        assertThat(payload.getChangedFields()).hasSize(1);
        EventChangedFieldCreateDto changedField = payload.getChangedFields().get(0);
        assertThat(changedField.getId()).isEqualTo("status");
        assertThat(changedField.getOldValue()).isEqualTo("Черновик");
        assertThat(changedField.getNewValue()).isEqualTo("На согласовании включения");

        OutboxMessageEvent event = eventCaptor.getValue();
        assertThat(event.getUuid()).isEqualTo(outboxMessageUuid);
    }

    @Test
    void givenOutboxServiceThrows_whenSendToApproveEvent_thenPropagateExceptionAndDoNotPublishEvent() {
        // given
        UUID entityUuid = UUID.randomUUID();
        ZonedDateTime updatedAt = ZonedDateTime.now();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(entityUuid);
        entity.setVehicleNumber("A123AA777");
        entity.setVehicleBrand("КАМАЗ");
        entity.setStatusId(StsStatus.TO_APPROVE_OUT);
        entity.setUpdatedBy(AUTHOR_UUID);
        entity.setUpdatedAt(updatedAt);

        mockSecurityContext(FULL_NAME);

        OutboxMessageActionException exception =
                new OutboxMessageActionException(
                        "Unable to serialize payload for outbox message",
                        new RuntimeException()
                );

        doThrow(exception).when(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.CREATE_EVENT),
                eq(AUTHOR_UUID),
                eq(entityUuid.toString()),
                any(EventCreateDto.class)
        );

        // when / then
        assertThatThrownBy(() -> stsEventsHistoryService.sendToApproveEvent(entity, StsStatus.APPROVED))
                .isInstanceOf(OutboxMessageActionException.class)
                .hasMessage("Unable to serialize payload for outbox message");

        verify(outboxMessageService).addOutboxMessage(
                eq(OutboxMessageEventType.CREATE_EVENT),
                eq(AUTHOR_UUID),
                eq(entityUuid.toString()),
                any(EventCreateDto.class)
        );

        verifyNoInteractions(applicationEventPublisher);
        verifyNoMoreInteractions(outboxMessageService);
    }

    private void mockSecurityContext(String fullName) {
        AuthorizedUser authorizedUser = mock(AuthorizedUser.class);
        when(authorizedUser.getFullName()).thenReturn(fullName);

        Authentication authentication = mock(Authentication.class);
        when(authentication.getPrincipal()).thenReturn(authorizedUser);

        SecurityContext securityContext = mock(SecurityContext.class);
        when(securityContext.getAuthentication()).thenReturn(authentication);

        SecurityContextHolder.setContext(securityContext);
    }
}
```
