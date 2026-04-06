```java
@Test
void givenDraftEntity_whenToApprove_thenCalculateSaveAndSendEvents() {
    UUID uuid = UUID.randomUUID();

    StsDataEntity entity = new StsDataEntity();
    entity.setUuid(uuid);
    entity.setStatusId(StsStatus.DRAFT);

    when(stsDataService.getExistingByUuids(List.of(uuid)))
            .thenReturn(List.of(entity));

    when(stsStatusTransitionService.calculateNextStatus(entity, StsAction.TO_APPROVE))
            .thenReturn(StsStatus.TO_APPROVE_IN);

    when(stsDataService.saveAll(List.of(entity)))
            .thenReturn(List.of(entity));

    StsBatchOperationResult<StsDataEntity> result =
            stsWorkFlowService.toApprove(List.of(uuid));

    assertThat(result).isNotNull();
    assertThat(result.processed()).containsExactly(entity);
    assertThat(result.errors()).isEmpty();
    assertThat(entity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_IN);

    verify(stsDataService).getExistingByUuids(List.of(uuid));
    verify(stsStatusTransitionService)
            .calculateNextStatus(entity, StsAction.TO_APPROVE);
    verify(stsDataService).saveAll(List.of(entity));
    verify(stsEventsHistoryOutboxService)
            .sendToApproveEvent(entity, StsStatus.DRAFT);
    verify(stsTrackerHistoryOutboxService)
            .sendToApproveStatus(entity);

    verifyNoMoreInteractions(
            stsDataService,
            stsStatusTransitionService,
            stsEventsHistoryOutboxService,
            stsTrackerHistoryOutboxService
    );
}
```
