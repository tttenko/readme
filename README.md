```java
@Test
void givenApprovedEntity_whenToApprove_thenCalculateSaveAndSendEvents() {
    UUID uuid = UUID.randomUUID();

    StsDataEntity entity = new StsDataEntity();
    entity.setUuid(uuid);
    entity.setStatusId(StsStatus.APPROVED);

    when(stsDataService.getExistingByUuids(List.of(uuid)))
            .thenReturn(List.of(entity));

    when(stsStatusTransitionService.calculateNextStatus(entity, StsAction.TO_APPROVE))
            .thenReturn(StsStatus.TO_APPROVE_OUT);

    when(stsDataService.saveAll(List.of(entity)))
            .thenReturn(List.of(entity));

    StsBatchOperationResult<StsDataEntity> actualResult =
            stsWorkFlowService.toApprove(List.of(uuid));

    assertThat(actualResult).isNotNull();
    assertThat(actualResult.processed()).containsExactly(entity);
    assertThat(actualResult.errors()).isEmpty();
    assertThat(entity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_OUT);

    verify(stsDataService).getExistingByUuids(List.of(uuid));
    verify(stsStatusTransitionService)
            .calculateNextStatus(entity, StsAction.TO_APPROVE);
    verify(stsDataService).saveAll(List.of(entity));
    verify(stsEventsHistoryOutboxService)
            .sendToApproveEvent(entity, StsStatus.APPROVED);
    verify(stsTrackerHistoryOutboxService)
            .sendToApproveStatus(entity);

    verifyNoMoreInteractions(
            stsDataService,
            stsStatusTransitionService,
            stsEventsHistoryOutboxService,
            stsTrackerHistoryOutboxService
    );
}
@Test
void givenMultipleEntities_whenToApprove_thenUpdateEachEntityAndSendEvents() {
    UUID firstUuid = UUID.randomUUID();
    UUID secondUuid = UUID.randomUUID();

    StsDataEntity firstEntity = new StsDataEntity();
    firstEntity.setUuid(firstUuid);
    firstEntity.setStatusId(StsStatus.DRAFT);

    StsDataEntity secondEntity = new StsDataEntity();
    secondEntity.setUuid(secondUuid);
    secondEntity.setStatusId(StsStatus.APPROVED);

    List<StsDataEntity> entities = List.of(firstEntity, secondEntity);

    when(stsDataService.getExistingByUuids(List.of(firstUuid, secondUuid)))
            .thenReturn(entities);

    when(stsStatusTransitionService.calculateNextStatus(firstEntity, StsAction.TO_APPROVE))
            .thenReturn(StsStatus.TO_APPROVE_IN);
    when(stsStatusTransitionService.calculateNextStatus(secondEntity, StsAction.TO_APPROVE))
            .thenReturn(StsStatus.TO_APPROVE_OUT);

    when(stsDataService.saveAll(entities)).thenReturn(entities);

    StsBatchOperationResult<StsDataEntity> actualResult =
            stsWorkFlowService.toApprove(List.of(firstUuid, secondUuid));

    assertThat(actualResult).isNotNull();
    assertThat(actualResult.processed()).containsExactly(firstEntity, secondEntity);
    assertThat(actualResult.errors()).isEmpty();
    assertThat(firstEntity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_IN);
    assertThat(secondEntity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_OUT);

    verify(stsDataService).getExistingByUuids(List.of(firstUuid, secondUuid));
    verify(stsStatusTransitionService).calculateNextStatus(firstEntity, StsAction.TO_APPROVE);
    verify(stsStatusTransitionService).calculateNextStatus(secondEntity, StsAction.TO_APPROVE);
    verify(stsDataService).saveAll(entities);

    verify(stsEventsHistoryOutboxService).sendToApproveEvent(firstEntity, StsStatus.DRAFT);
    verify(stsEventsHistoryOutboxService).sendToApproveEvent(secondEntity, StsStatus.APPROVED);

    verify(stsTrackerHistoryOutboxService).sendToApproveStatus(firstEntity);
    verify(stsTrackerHistoryOutboxService).sendToApproveStatus(secondEntity);

    verifyNoMoreInteractions(
            stsDataService,
            stsStatusTransitionService,
            stsEventsHistoryOutboxService,
            stsTrackerHistoryOutboxService
    );
}
```
