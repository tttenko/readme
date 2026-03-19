```java

@ExtendWith(MockitoExtension.class)
class StsDataControllerImplTest {

    @Mock
    private StsDataService stsDataService;

    @Mock
    private StsDataMapper stsDataMapper;

    private StsDataControllerImpl stsDataController;

    @BeforeEach
    void setUp() {
        stsDataController = new StsDataControllerImpl(
                stsDataService,
                stsDataMapper
        );
    }

    @Test
    void givenValidRequest_whenCreateStsData_thenMapSetTechnicalFieldsSaveAndReturnWrappedDto() {
        // given
        UUID uuid = UUID.randomUUID();
        UUID contractUuid = UUID.randomUUID();

        CreateStsDataRequest request = new CreateStsDataRequest();
        request.setContractUuid(contractUuid);
        request.setTbCode("1234");
        request.setVehicleNumber("A123AA777");
        request.setVehicleBrand("КамАЗ");
        request.setComment("Тестовая запись");

        StsDataEntity mappedEntity = new StsDataEntity();
        mappedEntity.setContractUuid(contractUuid);
        mappedEntity.setTbCode("1234");
        mappedEntity.setVehicleNumber("A123AA777");
        mappedEntity.setVehicleBrand("КамАЗ");
        mappedEntity.setComment("Тестовая запись");

        StsDataEntity savedEntity = new StsDataEntity();
        savedEntity.setUuid(uuid);
        savedEntity.setContractUuid(contractUuid);
        savedEntity.setTbCode("1234");
        savedEntity.setVehicleNumber("A123AA777");
        savedEntity.setVehicleBrand("КамАЗ");
        savedEntity.setComment("Тестовая запись");
        savedEntity.setStatusId("01");
        savedEntity.setCreatedBy("system");
        savedEntity.setUpdatedBy("system");
        savedEntity.setDeleted(false);

        StsDataDto responseDto = new StsDataDto();
        responseDto.setUuid(uuid);
        responseDto.setContractUuid(contractUuid);
        responseDto.setTbCode("1234");
        responseDto.setVehicleNumber("A123AA777");
        responseDto.setVehicleBrand("КамАЗ");
        responseDto.setComment("Тестовая запись");
        responseDto.setStatusId("01");
        responseDto.setCreatedBy("system");
        responseDto.setUpdatedBy("system");
        responseDto.setDeleted(false);

        when(stsDataMapper.toEntity(request)).thenReturn(mappedEntity);
        when(stsDataService.create(mappedEntity)).thenReturn(savedEntity);
        when(stsDataMapper.toDto(savedEntity)).thenReturn(responseDto);

        // when
        ResultObj<StsDataDto> actualResult = stsDataController.createStsData(request);

        // then
        ArgumentCaptor<StsDataEntity> entityCaptor = ArgumentCaptor.forClass(StsDataEntity.class);
        verify(stsDataService).create(entityCaptor.capture());

        StsDataEntity entityForSave = entityCaptor.getValue();
        assertThat(entityForSave.getCreatedAt()).isNotNull();
        assertThat(entityForSave.getUpdatedAt()).isNotNull();
        assertThat(entityForSave.getCreatedBy()).isEqualTo("system");
        assertThat(entityForSave.getUpdatedBy()).isEqualTo("system");
        assertThat(entityForSave.getStatusId()).isEqualTo("01");
        assertThat(entityForSave.isDeleted()).isFalse();

        assertThat(actualResult).isNotNull();
        assertThat(actualResult.getData()).isSameAs(responseDto);
        assertThat(actualResult.getCount()).isEqualTo(1L);

        verify(stsDataMapper).toEntity(request);
        verify(stsDataService).create(mappedEntity);
        verify(stsDataMapper).toDto(savedEntity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper);
    }

    @Test
    void givenContractUuidAndPaging_whenGetStsDataByContractUuid_thenReturnWrappedMappedPageDto() {
        // given
        UUID contractUuid = UUID.randomUUID();
        Integer top = 10;
        Integer skip = 20;

        TopSkipOrderFilterParams params = new TopSkipOrderFilterParams();
        params.setTop(top);
        params.set$skip(skip);

        ZonedDateTime now = ZonedDateTime.now();

        StsDataEntity firstEntity = new StsDataEntity();
        firstEntity.setUuid(UUID.randomUUID());
        firstEntity.setContractUuid(contractUuid);
        firstEntity.setTbCode("1234");
        firstEntity.setVehicleNumber("A123AA777");
        firstEntity.setVehicleBrand("КамАЗ");
        firstEntity.setComment("Первая запись");
        firstEntity.setStatusId("01");
        firstEntity.setCreatedBy("user_1");
        firstEntity.setUpdatedBy("user_1");
        firstEntity.setCreatedAt(now);
        firstEntity.setUpdatedAt(now);
        firstEntity.setDeleted(false);

        StsDataEntity secondEntity = new StsDataEntity();
        secondEntity.setUuid(UUID.randomUUID());
        secondEntity.setContractUuid(contractUuid);
        secondEntity.setTbCode("5678");
        secondEntity.setVehicleNumber("B456BB777");
        secondEntity.setVehicleBrand("МАЗ");
        secondEntity.setComment("Вторая запись");
        secondEntity.setStatusId("02");
        secondEntity.setCreatedBy("user_2");
        secondEntity.setUpdatedBy("user_2");
        secondEntity.setCreatedAt(now);
        secondEntity.setUpdatedAt(now);
        secondEntity.setDeleted(false);

        StsDataDto firstDto = new StsDataDto();
        firstDto.setUuid(firstEntity.getUuid());
        firstDto.setContractUuid(contractUuid);
        firstDto.setTbCode("1234");

        StsDataDto secondDto = new StsDataDto();
        secondDto.setUuid(secondEntity.getUuid());
        secondDto.setContractUuid(contractUuid);
        secondDto.setTbCode("5678");

        Page<StsDataEntity> page = new PageImpl<>(
                List.of(firstEntity, secondEntity),
                PageRequest.of(skip / top, top, Sort.by(Sort.Direction.DESC, "createdAt")),
                35
        );

        when(stsDataService.getPageByContractUuid(contractUuid, params)).thenReturn(page);
        when(stsDataMapper.toDto(firstEntity)).thenReturn(firstDto);
        when(stsDataMapper.toDto(secondEntity)).thenReturn(secondDto);

        // when
        ResultObj<List<StsDataDto>> actualResult =
                stsDataController.getStsDataByContractUuid(contractUuid, params);

        // then
        assertThat(actualResult).isNotNull();
        assertThat(actualResult.getData()).hasSize(2);
        assertThat(actualResult.getData().get(0)).isSameAs(firstDto);
        assertThat(actualResult.getData().get(1)).isSameAs(secondDto);
        assertThat(actualResult.getCount()).isEqualTo(35L);

        verify(stsDataService).getPageByContractUuid(contractUuid, params);
        verify(stsDataMapper).toDto(firstEntity);
        verify(stsDataMapper).toDto(secondEntity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper);
    }

    @Test
    void givenUuid_whenGetStsDataByUuid_thenReturnWrappedDto() {
        // given
        UUID uuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(uuid);

        StsDataDto dto = new StsDataDto();
        dto.setUuid(uuid);

        when(stsDataService.getByUuid(uuid)).thenReturn(entity);
        when(stsDataMapper.toDto(entity)).thenReturn(dto);

        // when
        ResultObj<StsDataDto> actualResult = stsDataController.getStsDataByUuid(uuid);

        // then
        assertThat(actualResult).isNotNull();
        assertThat(actualResult.getData()).isSameAs(dto);
        assertThat(actualResult.getCount()).isEqualTo(1L);

        verify(stsDataService).getByUuid(uuid);
        verify(stsDataMapper).toDto(entity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper);
    }
}
```
