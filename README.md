```java
@Test
void givenOutboxServiceThrowsOutboxMessageActionException_whenSendCreateEvent_thenPropagateExceptionAndDoNotPublishEvent() {
    // given
    UUID entityUuid = UUID.randomUUID();
    UUID contractUuid = UUID.randomUUID();
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

    AuthorizedUser authorizedUser = mock(AuthorizedUser.class);
    when(authorizedUser.getFullName()).thenReturn(FULL_NAME);

    Authentication authentication = mock(Authentication.class);
    when(authentication.getPrincipal()).thenReturn(authorizedUser);

    SecurityContext securityContext = mock(SecurityContext.class);
    when(securityContext.getAuthentication()).thenReturn(authentication);

    SecurityContextHolder.setContext(securityContext);

    OutboxMessageActionException exception =
            new OutboxMessageActionException("Unable to serialize payload for outbox message", new RuntimeException());

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
```
