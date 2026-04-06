```java
@Test
void givenDraftEntity_whenToApprove_thenMoveToApproveInSaveAndSendEvents() {
    // given
    UUID uuid = UUID.randomUUID();

    StsDataEntity entity = new StsDataEntity();
    entity.setUuid(uuid);
    entity.setStatusId(StsStatus.DRAFT);

    when(stsDataRepository.findAllByUuidInAndDeleted(List.of(uuid), false))
            .thenReturn(List.of(entity));
    when(stsDataRepository.saveAll(List.of(entity)))
            .thenReturn(List.of(entity));

    // when
    List<StsDataEntity> actualEntities = stsDataService.toApprove(List.of(uuid));

    // then
    assertThat(actualEntities).containsExactly(entity);
    assertThat(entity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_IN);

    verify(stsDataRepository).findAllByUuidInAndDeleted(List.of(uuid), false);
    verify(stsDataRepository).saveAll(List.of(entity));
    verify(stsEventsHistoryOutboxService).sendToApproveEvent(entity, StsStatus.DRAFT);
    verify(stsTrackerHistoryOutboxService).sendToApproveStatus(entity);

    verifyNoMoreInteractions(
            stsDataRepository,
            stsEventsHistoryOutboxService,
            stsTrackerHistoryOutboxService
    );
}

@Test
void givenApprovedEntity_whenToApprove_thenMoveToApproveOutSaveAndSendEvents() {
    // given
    UUID uuid = UUID.randomUUID();

    StsDataEntity entity = new StsDataEntity();
    entity.setUuid(uuid);
    entity.setStatusId(StsStatus.APPROVED);

    when(stsDataRepository.findAllByUuidInAndDeleted(List.of(uuid), false))
            .thenReturn(List.of(entity));
    when(stsDataRepository.saveAll(List.of(entity)))
            .thenReturn(List.of(entity));

    // when
    List<StsDataEntity> actualEntities = stsDataService.toApprove(List.of(uuid));

    // then
    assertThat(actualEntities).containsExactly(entity);
    assertThat(entity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_OUT);

    verify(stsDataRepository).findAllByUuidInAndDeleted(List.of(uuid), false);
    verify(stsDataRepository).saveAll(List.of(entity));
    verify(stsEventsHistoryOutboxService).sendToApproveEvent(entity, StsStatus.APPROVED);
    verify(stsTrackerHistoryOutboxService).sendToApproveStatus(entity);

    verifyNoMoreInteractions(
            stsDataRepository,
            stsEventsHistoryOutboxService,
            stsTrackerHistoryOutboxService
    );
}

@Test
void givenMissingUuid_whenToApprove_thenThrowEntityNotFoundException() {
    // given
    UUID existingUuid = UUID.randomUUID();
    UUID missingUuid = UUID.randomUUID();

    StsDataEntity entity = new StsDataEntity();
    entity.setUuid(existingUuid);
    entity.setStatusId(StsStatus.DRAFT);

    when(stsDataRepository.findAllByUuidInAndDeleted(List.of(existingUuid, missingUuid), false))
            .thenReturn(List.of(entity));

    // when / then
    assertThatThrownBy(() -> stsDataService.toApprove(List.of(existingUuid, missingUuid)))
            .isInstanceOf(EntityNotFoundException.class)
            .hasMessage("Записи СТС не найдены по uuid: [" + missingUuid + "]");

    verify(stsDataRepository).findAllByUuidInAndDeleted(List.of(existingUuid, missingUuid), false);
    verifyNoMoreInteractions(
            stsDataRepository,
            stsEventsHistoryOutboxService,
            stsTrackerHistoryOutboxService
    );
}

@Test
void givenUnsupportedStatus_whenToApprove_thenThrowIllegalStateException() {
    // given
    UUID uuid = UUID.randomUUID();

    StsDataEntity entity = new StsDataEntity();
    entity.setUuid(uuid);
    entity.setStatusId(StsStatus.TO_APPROVE_IN);

    when(stsDataRepository.findAllByUuidInAndDeleted(List.of(uuid), false))
            .thenReturn(List.of(entity));

    // when / then
    assertThatThrownBy(() -> stsDataService.toApprove(List.of(uuid)))
            .isInstanceOf(IllegalStateException.class)
            .hasMessage("Операция toApprove недоступна для статуса: На согласовании включения");

    verify(stsDataRepository).findAllByUuidInAndDeleted(List.of(uuid), false);
    verifyNoMoreInteractions(
            stsDataRepository,
            stsEventsHistoryOutboxService,
            stsTrackerHistoryOutboxService
    );
}

Если хочешь покрыть еще и массовый сценарий, можно добавить пятый тест — смешанный список из DRAFT и APPROVED:

@Test
void givenDraftAndApprovedEntities_whenToApprove_thenUpdateEachEntityAndSendEvents() {
    // given
    UUID firstUuid = UUID.randomUUID();
    UUID secondUuid = UUID.randomUUID();

    StsDataEntity firstEntity = new StsDataEntity();
    firstEntity.setUuid(firstUuid);
    firstEntity.setStatusId(StsStatus.DRAFT);

    StsDataEntity secondEntity = new StsDataEntity();
    secondEntity.setUuid(secondUuid);
    secondEntity.setStatusId(StsStatus.APPROVED);

    when(stsDataRepository.findAllByUuidInAndDeleted(List.of(firstUuid, secondUuid), false))
            .thenReturn(List.of(firstEntity, secondEntity));
    when(stsDataRepository.saveAll(List.of(firstEntity, secondEntity)))
            .thenReturn(List.of(firstEntity, secondEntity));

    // when
    List<StsDataEntity> actualEntities = stsDataService.toApprove(List.of(firstUuid, secondUuid));

    // then
    assertThat(actualEntities).containsExactly(firstEntity, secondEntity);
    assertThat(firstEntity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_IN);
    assertThat(secondEntity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_OUT);

    verify(stsDataRepository).findAllByUuidInAndDeleted(List.of(firstUuid, secondUuid), false);
    verify(stsDataRepository).saveAll(List.of(firstEntity, secondEntity));

    verify(stsEventsHistoryOutboxService).sendToApproveEvent(firstEntity, StsStatus.DRAFT);
    verify(stsEventsHistoryOutboxService).sendToApproveEvent(secondEntity, StsStatus.APPROVED);

    verify(stsTrackerHistoryOutboxService).sendToApproveStatus(firstEntity);
    verify(stsTrackerHistoryOutboxService).sendToApproveStatus(secondEntity);

    verifyNoMoreInteractions(
            stsDataRepository,
            stsEventsHistoryOutboxService,
            stsTrackerHistoryOutboxService
    );
}
```
