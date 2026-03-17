```java

@ExtendWith(MockitoExtension.class)
class StsDataServiceImplTest {

    @Mock
    private StsDataRepository stsDataRepository;

    @Mock
    private StsDataMapper stsDataMapper;

    @InjectMocks
    private StsDataServiceImpl stsDataService;

    @Test
    void givenValidRequest_whenCreate_thenReturnDto() {
        // given
        CreateStsDataRequest request = new CreateStsDataRequest();
        request.setContractUuid(UUID.randomUUID());
        request.setTbId("1234");
        request.setVehicleNumber("A123AA777");
        request.setVehicleBrand("КамАЗ");
        request.setComment("Тестовая запись");
        request.setStatusId("01");
        request.setCreatedBy("supplier_user");

        StsDataEntity mappedEntity = new StsDataEntity();
        mappedEntity.setContractUuid(request.getContractUuid());
        mappedEntity.setTbId(request.getTbId());
        mappedEntity.setVehicleNumber(request.getVehicleNumber());
        mappedEntity.setVehicleBrand(request.getVehicleBrand());
        mappedEntity.setComment(request.getComment());
        mappedEntity.setStatusId(request.getStatusId());
        mappedEntity.setCreatedBy(request.getCreatedBy());

        UUID savedUuid = UUID.randomUUID();
        StsDataEntity savedEntity = new StsDataEntity();
        savedEntity.setUuid(savedUuid);
        savedEntity.setContractUuid(request.getContractUuid());
        savedEntity.setTbId(request.getTbId());
        savedEntity.setVehicleNumber(request.getVehicleNumber());
        savedEntity.setVehicleBrand(request.getVehicleBrand());
        savedEntity.setComment(request.getComment());
        savedEntity.setStatusId(request.getStatusId());
        savedEntity.setCreatedBy(request.getCreatedBy());
        savedEntity.setUpdatedBy(request.getCreatedBy());
        savedEntity.setDeleted(false);

        StsDataDto expectedDto = new StsDataDto();
        expectedDto.setUuid(savedUuid);
        expectedDto.setContractUuid(request.getContractUuid());
        expectedDto.setTbId(request.getTbId());
        expectedDto.setVehicleNumber(request.getVehicleNumber());
        expectedDto.setVehicleBrand(request.getVehicleBrand());
        expectedDto.setComment(request.getComment());
        expectedDto.setStatusId(request.getStatusId());
        expectedDto.setCreatedBy(request.getCreatedBy());
        expectedDto.setChangedBy(request.getCreatedBy());
        expectedDto.setDeleted(false);

        when(stsDataMapper.toEntity(request)).thenReturn(mappedEntity);
        when(stsDataRepository.save(mappedEntity)).thenReturn(savedEntity);
        when(stsDataMapper.toDto(savedEntity)).thenReturn(expectedDto);

        // when
        StsDataDto actualDto = stsDataService.create(request);

        // then
        ArgumentCaptor<StsDataEntity> entityCaptor = ArgumentCaptor.forClass(StsDataEntity.class);
        verify(stsDataRepository).save(entityCaptor.capture());
        StsDataEntity entityForSave = entityCaptor.getValue();

        assertThat(entityForSave.getCreatedAt()).isNotNull();
        assertThat(entityForSave.getUpdatedAt()).isNotNull();
        assertThat(entityForSave.getCreatedAt()).isEqualTo(entityForSave.getUpdatedAt());
        assertThat(entityForSave.getUpdatedBy()).isEqualTo(request.getCreatedBy());
        assertThat(entityForSave.isDeleted()).isFalse();

        assertThat(actualDto).isSameAs(expectedDto);

        verify(stsDataMapper).toEntity(request);
        verify(stsDataRepository).save(mappedEntity);
        verify(stsDataMapper).toDto(savedEntity);
        verifyNoMoreInteractions(stsDataRepository, stsDataMapper);
    }

    @Test
    void givenExistingData_whenGetByContractUuid_thenReturnDtoList() {
        // given
        UUID contractUuid = UUID.randomUUID();

        StsDataEntity firstEntity = new StsDataEntity();
        firstEntity.setUuid(UUID.randomUUID());

        StsDataEntity secondEntity = new StsDataEntity();
        secondEntity.setUuid(UUID.randomUUID());

        StsDataDto firstDto = new StsDataDto();
        firstDto.setUuid(firstEntity.getUuid());

        StsDataDto secondDto = new StsDataDto();
        secondDto.setUuid(secondEntity.getUuid());

        when(stsDataRepository.findAllByContractUuidAndDeletedOrderByCreatedAtDesc(contractUuid, false))
                .thenReturn(List.of(firstEntity, secondEntity));
        when(stsDataMapper.toDto(firstEntity)).thenReturn(firstDto);
        when(stsDataMapper.toDto(secondEntity)).thenReturn(secondDto);

        // when
        List<StsDataDto> actualResult = stsDataService.getByContractUuid(contractUuid);

        // then
        assertThat(actualResult)
                .isNotNull()
                .hasSize(2)
                .containsExactly(firstDto, secondDto);

        verify(stsDataRepository).findAllByContractUuidAndDeletedOrderByCreatedAtDesc(contractUuid, false);
        verify(stsDataMapper).toDto(firstEntity);
        verify(stsDataMapper).toDto(secondEntity);
        verifyNoMoreInteractions(stsDataRepository, stsDataMapper);
    }

    @Test
    void givenNoData_whenGetByContractUuid_thenReturnEmptyList() {
        // given
        UUID contractUuid = UUID.randomUUID();

        when(stsDataRepository.findAllByContractUuidAndDeletedOrderByCreatedAtDesc(contractUuid, false))
                .thenReturn(List.of());

        // when
        List<StsDataDto> actualResult = stsDataService.getByContractUuid(contractUuid);

        // then
        assertThat(actualResult).isNotNull().isEmpty();

        verify(stsDataRepository).findAllByContractUuidAndDeletedOrderByCreatedAtDesc(contractUuid, false);
        verifyNoInteractions(stsDataMapper);
        verifyNoMoreInteractions(stsDataRepository);
    }

    @Test
    void givenExistingUuid_whenGetByUuid_thenReturnDto() {
        // given
        UUID uuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(uuid);

        StsDataDto expectedDto = new StsDataDto();
        expectedDto.setUuid(uuid);

        when(stsDataRepository.findByUuidAndDeleted(uuid, false)).thenReturn(Optional.of(entity));
        when(stsDataMapper.toDto(entity)).thenReturn(expectedDto);

        // when
        StsDataDto actualDto = stsDataService.getByUuid(uuid);

        // then
        assertThat(actualDto).isSameAs(expectedDto);

        verify(stsDataRepository).findByUuidAndDeleted(uuid, false);
        verify(stsDataMapper).toDto(entity);
        verifyNoMoreInteractions(stsDataRepository, stsDataMapper);
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
        verifyNoInteractions(stsDataMapper);
        verifyNoMoreInteractions(stsDataRepository);
    }
}
```
