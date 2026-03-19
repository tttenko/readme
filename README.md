```java
@ExtendWith(MockitoExtension.class)
class StsDataServiceTest {

    @Mock
    private StsDataRepository stsDataRepository;

    @InjectMocks
    private StsDataService stsDataService;

    @Test
    void givenValidEntity_whenCreate_thenReturnSavedEntity() {
        // given
        StsDataEntity entity = new StsDataEntity();
        entity.setContractUuid(UUID.randomUUID());
        entity.setTbCode("1234");
        entity.setVehicleNumber("A123AA777");
        entity.setVehicleBrand("КамАЗ");
        entity.setComment("Тестовая запись");
        entity.setStatusId("01");
        entity.setCreatedBy("system");

        UUID savedUuid = UUID.randomUUID();
        StsDataEntity savedEntity = new StsDataEntity();
        savedEntity.setUuid(savedUuid);
        savedEntity.setContractUuid(entity.getContractUuid());
        savedEntity.setTbCode(entity.getTbCode());
        savedEntity.setVehicleNumber(entity.getVehicleNumber());
        savedEntity.setVehicleBrand(entity.getVehicleBrand());
        savedEntity.setComment(entity.getComment());
        savedEntity.setStatusId(entity.getStatusId());
        savedEntity.setCreatedBy(entity.getCreatedBy());

        when(stsDataRepository.save(entity)).thenReturn(savedEntity);

        // when
        StsDataEntity actualEntity = stsDataService.create(entity);

        // then
        assertThat(actualEntity).isSameAs(savedEntity);

        verify(stsDataRepository).save(entity);
        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenExistingData_whenGetPageByContractUuid_thenReturnPage() {
        // given
        UUID contractUuid = UUID.randomUUID();

        TopSkipOrderFilterParams parameters = new TopSkipOrderFilterParams();
        parameters.setTop(10);
        parameters.setSkip(20);

        StsDataEntity firstEntity = new StsDataEntity();
        firstEntity.setUuid(UUID.randomUUID());

        StsDataEntity secondEntity = new StsDataEntity();
        secondEntity.setUuid(UUID.randomUUID());

        Page<StsDataEntity> expectedPage = new PageImpl<>(
                List.of(firstEntity, secondEntity),
                PageRequest.of(2, 10, Sort.by(Sort.Direction.DESC, "createdAt")),
                2
        );

        when(stsDataRepository.findAllByContractUuidAndDeleted(eq(contractUuid), eq(false), any(Pageable.class)))
                .thenReturn(expectedPage);

        // when
        Page<StsDataEntity> actualPage = stsDataService.getPageByContractUuid(contractUuid, parameters);

        // then
        assertThat(actualPage).isNotNull();
        assertThat(actualPage.getContent())
                .hasSize(2)
                .containsExactly(firstEntity, secondEntity);

        ArgumentCaptor<Pageable> pageableCaptor = ArgumentCaptor.forClass(Pageable.class);
        verify(stsDataRepository).findAllByContractUuidAndDeleted(eq(contractUuid), eq(false), pageableCaptor.capture());

        Pageable actualPageable = pageableCaptor.getValue();
        assertThat(actualPageable.getPageNumber()).isEqualTo(2);
        assertThat(actualPageable.getPageSize()).isEqualTo(10);
        assertThat(actualPageable.getSort()).isEqualTo(Sort.by(Sort.Direction.DESC, "createdAt"));

        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenNoData_whenGetPageByContractUuid_thenReturnEmptyPage() {
        // given
        UUID contractUuid = UUID.randomUUID();

        TopSkipOrderFilterParams parameters = new TopSkipOrderFilterParams();
        parameters.setTop(10);
        parameters.setSkip(0);

        Page<StsDataEntity> emptyPage = Page.empty(
                PageRequest.of(0, 10, Sort.by(Sort.Direction.DESC, "createdAt"))
        );

        when(stsDataRepository.findAllByContractUuidAndDeleted(eq(contractUuid), eq(false), any(Pageable.class)))
                .thenReturn(emptyPage);

        // when
        Page<StsDataEntity> actualPage = stsDataService.getPageByContractUuid(contractUuid, parameters);

        // then
        assertThat(actualPage).isNotNull();
        assertThat(actualPage.getContent()).isEmpty();
        assertThat(actualPage.getTotalElements()).isZero();

        ArgumentCaptor<Pageable> pageableCaptor = ArgumentCaptor.forClass(Pageable.class);
        verify(stsDataRepository).findAllByContractUuidAndDeleted(eq(contractUuid), eq(false), pageableCaptor.capture());

        Pageable actualPageable = pageableCaptor.getValue();
        assertThat(actualPageable.getPageNumber()).isEqualTo(0);
        assertThat(actualPageable.getPageSize()).isEqualTo(10);
        assertThat(actualPageable.getSort()).isEqualTo(Sort.by(Sort.Direction.DESC, "createdAt"));

        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenExistingUuid_whenGetByUuid_thenReturnEntity() {
        // given
        UUID uuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(uuid);

        when(stsDataRepository.findByUuidAndDeleted(uuid, false)).thenReturn(Optional.of(entity));

        // when
        StsDataEntity actualEntity = stsDataService.getByUuid(uuid);

        // then
        assertThat(actualEntity).isSameAs(entity);

        verify(stsDataRepository).findByUuidAndDeleted(uuid, false);
        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenMissingUuid_whenGetByUuid_thenThrowException() {
        // given
        UUID uuid = UUID.randomUUID();

        when(stsDataRepository.findByUuidAndDeleted(uuid, false)).thenReturn(Optional.empty());

        // when / then
        assertThatThrownBy(() -> stsDataService.getByUuid(uuid))
                .isInstanceOf(EntityNotFoundException.class)
                .hasMessage("Запись СТС не найдена по uuid: " + uuid);

        verify(stsDataRepository).findByUuidAndDeleted(uuid, false);
        verifyNoMoreInteractions(stsDataRepository);
    }
}
```
