```java
@Test
void givenValidEntity_whenSendCreateEvent_thenSaveMessageToOutboxAndPublishEvent() {
    // given
    UUID entityUuid = UUID.randomUUID();
    UUID contractUuid = UUID.randomUUID();
    UUID outboxMessageUuid = UUID.randomUUID();
    ZonedDateTime createdAt = ZonedDateTime.now();

    StsDataEntity entity = new StsDataEntity();
    entity.setUuid(entityUuid);
    entity.setContractUuid(contractUuid);
    entity.setTbCode("1234");
    entity.setVehicleNumber("A123AA777");
    entity.setVehicleBrand("KAMA3");
    entity.setComment("Тестовая запись");
    entity.setStatusId(StsStatus.DRAFT);
    entity.setCreatedBy(AUTHOR_UUID);
    entity.setCreatedAt(createdAt);

    OutboxMessage outboxMessage = mock(OutboxMessage.class);
    when(outboxMessage.getUuid()).thenReturn(outboxMessageUuid);

    when(outboxMessageService.addOutboxMessage(
            eq(OutboxMessageEventType.CREATE_EVENT),
            eq(AUTHOR_UUID),
            eq(entityUuid.toString()),
            any(EventCreateDto.class)
    )).thenReturn(outboxMessage);

    AuthorizedUser authorizedUser = mock(AuthorizedUser.class);
    when(authorizedUser.getFullName()).thenReturn(FULL_NAME);

    Authentication authentication = mock(Authentication.class);
    when(authentication.getPrincipal()).thenReturn(authorizedUser);

    SecurityContext securityContext = mock(SecurityContext.class);
    when(securityContext.getAuthentication()).thenReturn(authentication);

    SecurityContextHolder.setContext(securityContext);

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

    EventCreateDto actualPayload = payloadCaptor.getValue();
    assertThat(actualPayload).isNotNull();
    assertThat(actualPayload.getServiceId()).isEqualTo(StsHistoryEventIds.SERVICE_ID);
    assertThat(actualPayload.getId()).isEqualTo(StsHistoryEventIds.STS_CREATE);
    assertThat(actualPayload.getEntityUuid()).isEqualTo(entityUuid);
    assertThat(actualPayload.getSubmittedAt()).isEqualTo(createdAt);
    assertThat(actualPayload.getSubmittedBy()).isEqualTo(AUTHOR_UUID.toString());
    assertThat(actualPayload.getUserName()).isEqualTo(FULL_NAME);
    assertThat(actualPayload.getSession()).isEqualTo(NO_SESSION);
    assertThat(actualPayload.getUserNode()).isEqualTo(NO_USER_NODE);

    assertThat(actualPayload.getParameters()).isNotNull();
    assertThat(actualPayload.getParameters()).hasSize(2);

    Map<String, String> parameters = actualPayload.getParameters().stream()
            .collect(Collectors.toMap(
                    EventParameterCreateDto::getId,
                    EventParameterCreateDto::getValue
            ));

    assertThat(parameters.get("vehicle_num")).isEqualTo("A123AA777");
    assertThat(parameters.get("vehicle_name")).isEqualTo("KAMA3");

    OutboxMessageEvent actualEvent = eventCaptor.getValue();
    assertThat(actualEvent).isNotNull();
    assertThat(actualEvent.uuid()).isEqualTo(outboxMessageUuid);
}

```
