```java
@Test
    void givenValidEntity_whenCreate_thenReturnSavedEntity() {
        // given
        UUID authorUuid = UUID.fromString("00000000-0000-0000-0000-000000000001");

        StsDataEntity entity = new StsDataEntity();
        entity.setContractUuid(UUID.randomUUID());
        entity.setTbCode("1234");
        entity.setVehicleNumber("A123AA777");
        entity.setVehicleBrand("КамАЗ");
        entity.setComment("Тестовая запись");
        entity.setCreatedBy(authorUuid);

        UUID savedUuid = UUID.randomUUID();

        StsDataEntity savedEntity = new StsDataEntity();
        savedEntity.setUuid(savedUuid);
        savedEntity.setContractUuid(entity.getContractUuid());
        savedEntity.setTbCode(entity.getTbCode());
        savedEntity.setVehicleNumber(entity.getVehicleNumber());
        savedEntity.setVehicleBrand(entity.getVehicleBrand());
        savedEntity.setComment(entity.getComment());
        savedEntity.setStatusId(StsStatus.DRAFT);
        savedEntity.setCreatedBy(authorUuid);
        savedEntity.setDeleted(false);

        when(stsDataRepository.save(entity)).thenReturn(savedEntity);

        // when
        StsDataEntity actualEntity = stsDataService.create(entity);

        // then
        assertThat(actualEntity).isSameAs(savedEntity);

        assertThat(entity.getStatusId()).isEqualTo(StsStatus.DRAFT);
        assertThat(entity.isDeleted()).isFalse();

        verify(stsDataRepository).save(entity);
        verify(stsEventsHistoryService).sendCreateEvent(savedEntity);
        verify(applicationEventPublisher)
                .publishEvent(StsCreatedTrackerHistoryEvent.from(savedEntity));

        verifyNoMoreInteractions(
                stsDataRepository,
                stsEventsHistoryService,
                applicationEventPublisher
        );
    }
```
