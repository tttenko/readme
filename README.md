```java
@Test
void sendCreatedStatus_shouldPublishOutboxMessageEventWithReturnedMessageUuid() {
    UUID entityUuid = UUID.randomUUID();
    UUID createdBy = UUID.randomUUID();
    UUID outboxUuid = UUID.randomUUID();

    StsDataEntity entity = new StsDataEntity();
    entity.setUuid(entityUuid);
    entity.setCreatedBy(createdBy);
    entity.setStatusId(StsStatus.DRAFT);

    OutboxMessage outboxMessage = mock(OutboxMessage.class);
    when(outboxMessage.getUuid()).thenReturn(outboxUuid);
    when(outboxMessageService.addOutboxMessage(any(), any(), any(), any())).thenReturn(outboxMessage);

    stsTrackerHistoryOutboxService.sendCreatedStatus(entity);

    verify(applicationEventPublisher).publishEvent(new OutboxMessageEvent(outboxUuid));
}
```
