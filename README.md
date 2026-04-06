```java
@ExtendWith(MockitoExtension.class)
class StsDataServiceTest {

    @Mock
    private StsDataRepository stsDataRepository;

    @InjectMocks
    private StsDataService stsDataService;

    @Test
    void givenExistingData_whenGetPageByContractUuid_thenReturnPage() {
        UUID contractUuid = UUID.randomUUID();

        TopSkipOrderFilterParams parameters = new TopSkipOrderFilterParams();
        parameters.setTop(10);
        parameters.setSkip(20);
        parameters.setOrderby("createdAt desc");

        StsDataEntity firstEntity = new StsDataEntity();
        firstEntity.setUuid(UUID.randomUUID());

        StsDataEntity secondEntity = new StsDataEntity();
        secondEntity.setUuid(UUID.randomUUID());

        Page<StsDataEntity> expectedPage = new PageImpl<>(
                List.of(firstEntity, secondEntity),
                PageRequest.of(2, 10, Sort.by(Sort.Direction.DESC, "createdAt")),
                2
        );

        when(stsDataRepository.findAll(any(Specification.class), any(Pageable.class)))
                .thenReturn(expectedPage);

        Page<StsDataEntity> actualPage =
                stsDataService.getPageByContractUuid(contractUuid, parameters);

        assertThat(actualPage).isNotNull();
        assertThat(actualPage.getContent())
                .hasSize(2)
                .containsExactly(firstEntity, secondEntity);

        ArgumentCaptor<Specification<StsDataEntity>> specificationCaptor =
                ArgumentCaptor.forClass(Specification.class);
        ArgumentCaptor<Pageable> pageableCaptor =
                ArgumentCaptor.forClass(Pageable.class);

        verify(stsDataRepository)
                .findAll(specificationCaptor.capture(), pageableCaptor.capture());

        assertThat(specificationCaptor.getValue()).isNotNull();

        Pageable actualPageable = pageableCaptor.getValue();
        assertThat(actualPageable.getPageNumber()).isEqualTo(2);
        assertThat(actualPageable.getPageSize()).isEqualTo(10);
        assertThat(actualPageable.getSort())
                .isEqualTo(Sort.by(Sort.Direction.DESC, "createdAt"));

        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenNoData_whenGetPageByContractUuid_thenReturnEmptyPage() {
        UUID contractUuid = UUID.randomUUID();

        TopSkipOrderFilterParams parameters = new TopSkipOrderFilterParams();
        parameters.setTop(10);
        parameters.setSkip(0);
        parameters.setOrderby("createdAt desc");

        Page<StsDataEntity> emptyPage = Page.empty(
                PageRequest.of(0, 10, Sort.by(Sort.Direction.DESC, "createdAt"))
        );

        when(stsDataRepository.findAll(any(Specification.class), any(Pageable.class)))
                .thenReturn(emptyPage);

        Page<StsDataEntity> actualPage =
                stsDataService.getPageByContractUuid(contractUuid, parameters);

        assertThat(actualPage).isNotNull();
        assertThat(actualPage.getContent()).isEmpty();
        assertThat(actualPage.getTotalElements()).isZero();

        ArgumentCaptor<Specification<StsDataEntity>> specificationCaptor =
                ArgumentCaptor.forClass(Specification.class);
        ArgumentCaptor<Pageable> pageableCaptor =
                ArgumentCaptor.forClass(Pageable.class);

        verify(stsDataRepository)
                .findAll(specificationCaptor.capture(), pageableCaptor.capture());

        assertThat(specificationCaptor.getValue()).isNotNull();

        Pageable actualPageable = pageableCaptor.getValue();
        assertThat(actualPageable.getPageNumber()).isEqualTo(0);
        assertThat(actualPageable.getPageSize()).isEqualTo(10);
        assertThat(actualPageable.getSort())
                .isEqualTo(Sort.by(Sort.Direction.DESC, "createdAt"));

        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenExistingUuid_whenGetByUuid_thenReturnEntity() {
        UUID uuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(uuid);

        when(stsDataRepository.findByUuidAndDeleted(uuid, false))
                .thenReturn(Optional.of(entity));

        StsDataEntity actualEntity = stsDataService.getByUuid(uuid);

        assertThat(actualEntity).isSameAs(entity);

        verify(stsDataRepository).findByUuidAndDeleted(uuid, false);
        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenMissingUuid_whenGetByUuid_thenThrowException() {
        UUID uuid = UUID.randomUUID();

        when(stsDataRepository.findByUuidAndDeleted(uuid, false))
                .thenReturn(Optional.empty());

        assertThatThrownBy(() -> stsDataService.getByUuid(uuid))
                .isInstanceOf(EntityNotFoundException.class)
                .hasMessage("Запись СТС не найдена по uuid: " + uuid);

        verify(stsDataRepository).findByUuidAndDeleted(uuid, false);
        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenExistingUuids_whenGetByUuids_thenReturnEntities() {
        UUID firstUuid = UUID.randomUUID();
        UUID secondUuid = UUID.randomUUID();

        StsDataEntity firstEntity = new StsDataEntity();
        firstEntity.setUuid(firstUuid);

        StsDataEntity secondEntity = new StsDataEntity();
        secondEntity.setUuid(secondUuid);

        when(stsDataRepository.findAllByUuidInAndDeleted(List.of(firstUuid, secondUuid), false))
                .thenReturn(List.of(firstEntity, secondEntity));

        List<StsDataEntity> actualEntities = stsDataService.getByUuids(List.of(firstUuid, secondUuid));

        assertThat(actualEntities).containsExactly(firstEntity, secondEntity);

        verify(stsDataRepository).findAllByUuidInAndDeleted(List.of(firstUuid, secondUuid), false);
        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenMissingUuid_whenGetByUuids_thenThrowEntityNotFoundException() {
        UUID existingUuid = UUID.randomUUID();
        UUID missingUuid = UUID.randomUUID();

        List<UUID> uuids = List.of(existingUuid, missingUuid);

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(existingUuid);

        when(stsDataRepository.findAllByUuidInAndDeleted(uuids, false))
                .thenReturn(List.of(entity));

        assertThatThrownBy(() -> stsDataService.getByUuids(uuids))
                .isInstanceOf(EntityNotFoundException.class)
                .hasMessage("Записи СТС не найдены по uuid: [" + missingUuid + "]");

        verify(stsDataRepository).findAllByUuidInAndDeleted(uuids, false);
        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenDuplicateMissingUuids_whenGetByUuids_thenThrowEntityNotFoundExceptionWithoutDuplicates() {
        UUID existingUuid = UUID.randomUUID();
        UUID missingUuid = UUID.randomUUID();

        List<UUID> uuids = List.of(existingUuid, missingUuid, missingUuid);

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(existingUuid);

        when(stsDataRepository.findAllByUuidInAndDeleted(uuids, false))
                .thenReturn(List.of(entity));

        assertThatThrownBy(() -> stsDataService.getByUuids(uuids))
                .isInstanceOf(EntityNotFoundException.class)
                .hasMessage("Записи СТС не найдены по uuid: [" + missingUuid + "]");

        verify(stsDataRepository).findAllByUuidInAndDeleted(uuids, false);
        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenEntity_whenSave_thenReturnSavedEntity() {
        StsDataEntity entity = new StsDataEntity();
        StsDataEntity savedEntity = new StsDataEntity();

        when(stsDataRepository.save(entity)).thenReturn(savedEntity);

        StsDataEntity actualEntity = stsDataService.save(entity);

        assertThat(actualEntity).isSameAs(savedEntity);

        verify(stsDataRepository).save(entity);
        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenEntities_whenSaveAll_thenReturnSavedEntities() {
        StsDataEntity firstEntity = new StsDataEntity();
        StsDataEntity secondEntity = new StsDataEntity();
        List<StsDataEntity> entities = List.of(firstEntity, secondEntity);

        when(stsDataRepository.saveAll(entities)).thenReturn(entities);

        List<StsDataEntity> actualEntities = stsDataService.saveAll(entities);

        assertThat(actualEntities).containsExactly(firstEntity, secondEntity);

        verify(stsDataRepository).saveAll(entities);
        verifyNoMoreInteractions(stsDataRepository);
    }
}
StsWorkflowServiceTest
@ExtendWith(MockitoExtension.class)
class StsWorkflowServiceTest {

    @Mock
    private StsDataService stsDataService;

    @Mock
    private StsStatusTransitionService stsStatusTransitionService;

    @Mock
    private StsEventsHistoryOutboxService stsEventsHistoryOutboxService;

    @Mock
    private StsTrackerHistoryOutboxService stsTrackerHistoryOutboxService;

    @InjectMocks
    private StsWorkflowService stsWorkflowService;

    @Test
    void givenValidEntity_whenCreate_thenSetInitialFieldsSaveAndSendEvents() {
        UUID authorUuid = UUID.fromString("00000000-0000-0000-0000-000000000001");

        StsDataEntity entity = new StsDataEntity();
        entity.setContractUuid(UUID.randomUUID());
        entity.setTbCode("1234");
        entity.setVehicleNumber("A123AA777");
        entity.setVehicleBrand("КамАЗ");
        entity.setComment("Тестовая запись");
        entity.setCreatedBy(authorUuid);

        StsDataEntity savedEntity = new StsDataEntity();
        savedEntity.setUuid(UUID.randomUUID());
        savedEntity.setContractUuid(entity.getContractUuid());
        savedEntity.setTbCode(entity.getTbCode());
        savedEntity.setVehicleNumber(entity.getVehicleNumber());
        savedEntity.setVehicleBrand(entity.getVehicleBrand());
        savedEntity.setComment(entity.getComment());
        savedEntity.setStatusId(StsStatus.DRAFT);
        savedEntity.setCreatedBy(authorUuid);
        savedEntity.setDeleted(false);

        when(stsDataService.save(entity)).thenReturn(savedEntity);

        StsDataEntity actualEntity = stsWorkflowService.create(entity);

        assertThat(actualEntity).isSameAs(savedEntity);
        assertThat(entity.getStatusId()).isEqualTo(StsStatus.DRAFT);
        assertThat(entity.isDeleted()).isFalse();

        verify(stsDataService).save(entity);
        verify(stsEventsHistoryOutboxService).sendCreateEvent(savedEntity);
        verify(stsTrackerHistoryOutboxService).sendCreatedStatus(savedEntity);

        verifyNoMoreInteractions(
                stsDataService,
                stsStatusTransitionService,
                stsEventsHistoryOutboxService,
                stsTrackerHistoryOutboxService
        );
    }

    @Test
    void givenDraftEntity_whenToApprove_thenCalculateSaveAndSendEvents() {
        UUID uuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(uuid);
        entity.setStatusId(StsStatus.DRAFT);

        when(stsDataService.getByUuids(List.of(uuid))).thenReturn(List.of(entity));
        when(stsStatusTransitionService.calculateNextStatus(entity, StsAction.TO_APPROVE))
                .thenReturn(StsStatus.TO_APPROVE_IN);
        when(stsDataService.saveAll(List.of(entity))).thenReturn(List.of(entity));

        List<StsDataEntity> actualEntities = stsWorkflowService.toApprove(List.of(uuid));

        assertThat(actualEntities).containsExactly(entity);
        assertThat(entity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_IN);

        verify(stsDataService).getByUuids(List.of(uuid));
        verify(stsStatusTransitionService).calculateNextStatus(entity, StsAction.TO_APPROVE);
        verify(stsDataService).saveAll(List.of(entity));
        verify(stsEventsHistoryOutboxService).sendToApproveEvent(entity, StsStatus.DRAFT);
        verify(stsTrackerHistoryOutboxService).sendToApproveStatus(entity);

        verifyNoMoreInteractions(
                stsDataService,
                stsStatusTransitionService,
                stsEventsHistoryOutboxService,
                stsTrackerHistoryOutboxService
        );
    }

    @Test
    void givenApprovedEntity_whenToApprove_thenCalculateSaveAndSendEvents() {
        UUID uuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(uuid);
        entity.setStatusId(StsStatus.APPROVED);

        when(stsDataService.getByUuids(List.of(uuid))).thenReturn(List.of(entity));
        when(stsStatusTransitionService.calculateNextStatus(entity, StsAction.TO_APPROVE))
                .thenReturn(StsStatus.TO_APPROVE_OUT);
        when(stsDataService.saveAll(List.of(entity))).thenReturn(List.of(entity));

        List<StsDataEntity> actualEntities = stsWorkflowService.toApprove(List.of(uuid));

        assertThat(actualEntities).containsExactly(entity);
        assertThat(entity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_OUT);

        verify(stsDataService).getByUuids(List.of(uuid));
        verify(stsStatusTransitionService).calculateNextStatus(entity, StsAction.TO_APPROVE);
        verify(stsDataService).saveAll(List.of(entity));
        verify(stsEventsHistoryOutboxService).sendToApproveEvent(entity, StsStatus.APPROVED);
        verify(stsTrackerHistoryOutboxService).sendToApproveStatus(entity);

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

        when(stsDataService.getByUuids(List.of(firstUuid, secondUuid))).thenReturn(entities);
        when(stsStatusTransitionService.calculateNextStatus(firstEntity, StsAction.TO_APPROVE))
                .thenReturn(StsStatus.TO_APPROVE_IN);
        when(stsStatusTransitionService.calculateNextStatus(secondEntity, StsAction.TO_APPROVE))
                .thenReturn(StsStatus.TO_APPROVE_OUT);
        when(stsDataService.saveAll(entities)).thenReturn(entities);

        List<StsDataEntity> actualEntities = stsWorkflowService.toApprove(List.of(firstUuid, secondUuid));

        assertThat(actualEntities).containsExactly(firstEntity, secondEntity);
        assertThat(firstEntity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_IN);
        assertThat(secondEntity.getStatusId()).isEqualTo(StsStatus.TO_APPROVE_OUT);

        verify(stsDataService).getByUuids(List.of(firstUuid, secondUuid));
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

    @Test
    void givenTransitionServiceThrows_whenToApprove_thenDoNotSaveAndDoNotSendEvents() {
        UUID uuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(uuid);
        entity.setStatusId(StsStatus.TO_APPROVE_IN);

        when(stsDataService.getByUuids(List.of(uuid))).thenReturn(List.of(entity));
        when(stsStatusTransitionService.calculateNextStatus(entity, StsAction.TO_APPROVE))
                .thenThrow(new IllegalStateException("Ambiguous result"));

        assertThatThrownBy(() -> stsWorkflowService.toApprove(List.of(uuid)))
                .isInstanceOf(IllegalStateException.class)
                .hasMessage("Ambiguous result");

        verify(stsDataService).getByUuids(List.of(uuid));
        verify(stsStatusTransitionService).calculateNextStatus(entity, StsAction.TO_APPROVE);

        verifyNoMoreInteractions(
                stsDataService,
                stsStatusTransitionService,
                stsEventsHistoryOutboxService,
                stsTrackerHistoryOutboxService
        );
    }
}
StsStatusTransitionServiceTest

Подстрой import Node и Scheme под реальные классы из библиотеки.

@ExtendWith(MockitoExtension.class)
class StsStatusTransitionServiceTest {

    @Mock
    private Scheme stsTrackerScheme;

    @Mock
    private Node currentNode;

    @Mock
    private Node nextNode;

    @InjectMocks
    private StsStatusTransitionService stsStatusTransitionService;

    @Test
    void givenDraftEntityAndToApproveAction_whenCalculateNextStatus_thenReturnToApproveIn() {
        StsDataEntity entity = new StsDataEntity();
        entity.setStatusId(StsStatus.DRAFT);

        when(stsTrackerScheme.getNode(StsStatus.DRAFT.name())).thenReturn(currentNode);
        when(stsTrackerScheme.calculateNext(currentNode, Map.of("action", StsAction.TO_APPROVE.getId())))
                .thenReturn(nextNode);
        when(nextNode.getId()).thenReturn(StsStatus.TO_APPROVE_IN.name());

        StsStatus actualStatus =
                stsStatusTransitionService.calculateNextStatus(entity, StsAction.TO_APPROVE);

        assertThat(actualStatus).isEqualTo(StsStatus.TO_APPROVE_IN);

        verify(stsTrackerScheme).getNode(StsStatus.DRAFT.name());
        verify(stsTrackerScheme).calculateNext(currentNode, Map.of("action", StsAction.TO_APPROVE.getId()));
        verify(nextNode).getId();
        verifyNoMoreInteractions(stsTrackerScheme, currentNode, nextNode);
    }

    @Test
    void givenApprovedEntityAndToApproveAction_whenCalculateNextStatus_thenReturnToApproveOut() {
        StsDataEntity entity = new StsDataEntity();
        entity.setStatusId(StsStatus.APPROVED);

        when(stsTrackerScheme.getNode(StsStatus.APPROVED.name())).thenReturn(currentNode);
        when(stsTrackerScheme.calculateNext(currentNode, Map.of("action", StsAction.TO_APPROVE.getId())))
                .thenReturn(nextNode);
        when(nextNode.getId()).thenReturn(StsStatus.TO_APPROVE_OUT.name());

        StsStatus actualStatus =
                stsStatusTransitionService.calculateNextStatus(entity, StsAction.TO_APPROVE);

        assertThat(actualStatus).isEqualTo(StsStatus.TO_APPROVE_OUT);

        verify(stsTrackerScheme).getNode(StsStatus.APPROVED.name());
        verify(stsTrackerScheme).calculateNext(currentNode, Map.of("action", StsAction.TO_APPROVE.getId()));
        verify(nextNode).getId();
        verifyNoMoreInteractions(stsTrackerScheme, currentNode, nextNode);
    }

    @Test
    void givenSchemeReturnsUnknownStatus_whenCalculateNextStatus_thenThrowException() {
        StsDataEntity entity = new StsDataEntity();
        entity.setStatusId(StsStatus.DRAFT);

        when(stsTrackerScheme.getNode(StsStatus.DRAFT.name())).thenReturn(currentNode);
        when(stsTrackerScheme.calculateNext(currentNode, Map.of("action", StsAction.TO_APPROVE.getId())))
                .thenReturn(nextNode);
        when(nextNode.getId()).thenReturn("UNKNOWN_STATUS");

        assertThatThrownBy(() ->
                stsStatusTransitionService.calculateNextStatus(entity, StsAction.TO_APPROVE)
        ).isInstanceOf(IllegalArgumentException.class);

        verify(stsTrackerScheme).getNode(StsStatus.DRAFT.name());
        verify(stsTrackerScheme).calculateNext(currentNode, Map.of("action", StsAction.TO_APPROVE.getId()));
        verify(nextNode).getId();
        verifyNoMoreInteractions(stsTrackerScheme, currentNode, nextNode);
    }
}

```
