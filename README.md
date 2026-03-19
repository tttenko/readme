```java

@ExtendWith(MockitoExtension.class)
class StsDataControllerImplTest {

    @Mock
    private StsDataService stsDataService;

    @Mock
    private StsDataMapper stsDataMapper;

    private StsDataPageMapper stsDataPageMapper;
    private StsDataControllerImpl stsDataController;

    @BeforeEach
    void setUp() {
        stsDataPageMapper = new StsDataPageMapper(stsDataMapper);
        stsDataController = new StsDataControllerImpl(
                stsDataService,
                stsDataMapper,
                stsDataPageMapper
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
        request.setVehicleNumber("А123АА777");
        request.setVehicleBrand("КамАЗ");
        request.setComment("Тестовая запись");
        request.setStatusId("01");
        request.setCreatedBy("supplier_user");

        StsDataEntity mappedEntity = new StsDataEntity();
        mappedEntity.setContractUuid(contractUuid);
        mappedEntity.setTbCode("1234");
        mappedEntity.setVehicleNumber("А123АА777");
        mappedEntity.setVehicleBrand("КамАЗ");
        mappedEntity.setComment("Тестовая запись");
        mappedEntity.setStatusId("01");
        mappedEntity.setCreatedBy("supplier_user");

        StsDataEntity savedEntity = new StsDataEntity();
        savedEntity.setUuid(uuid);
        savedEntity.setContractUuid(contractUuid);
        savedEntity.setTbCode("1234");
        savedEntity.setVehicleNumber("А123АА777");
        savedEntity.setVehicleBrand("КамАЗ");
        savedEntity.setComment("Тестовая запись");
        savedEntity.setStatusId("01");
        savedEntity.setCreatedBy("supplier_user");
        savedEntity.setUpdatedBy("supplier_user");
        savedEntity.setDeleted(false);

        StsDataDto responseDto = new StsDataDto();
        responseDto.setUuid(uuid);
        responseDto.setContractUuid(contractUuid);
        responseDto.setTbCode("1234");
        responseDto.setVehicleNumber("А123АА777");
        responseDto.setVehicleBrand("КамАЗ");
        responseDto.setComment("Тестовая запись");
        responseDto.setStatusId("01");
        responseDto.setCreatedBy("supplier_user");
        responseDto.setUpdatedBy("supplier_user");
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
        assertThat(entityForSave.getCreatedAt()).isEqualTo(entityForSave.getUpdatedAt());
        assertThat(entityForSave.getUpdatedBy()).isEqualTo("supplier_user");
        assertThat(entityForSave.isDeleted()).isFalse();

        assertThat(actualResult).isNotNull();
        assertThat(actualResult.getData()).isSameAs(responseDto);

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

        ZonedDateTime now = ZonedDateTime.now();

        StsDataEntity firstEntity = new StsDataEntity();
        firstEntity.setUuid(UUID.randomUUID());
        firstEntity.setContractUuid(contractUuid);
        firstEntity.setTbCode("1234");
        firstEntity.setVehicleNumber("А123АА777");
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
        secondEntity.setVehicleNumber("В456ВВ777");
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
                PageRequest.of(2, 10, Sort.by(Sort.Direction.DESC, "createdAt")),
                35
        );

        when(stsDataService.getPageByContractUuid(contractUuid, top, skip)).thenReturn(page);
        when(stsDataMapper.toDto(firstEntity)).thenReturn(firstDto);
        when(stsDataMapper.toDto(secondEntity)).thenReturn(secondDto);

        // when
        ResultObj<PageDto<StsDataDto>> actualResult =
                stsDataController.getStsDataByContractUuid(contractUuid, null, null, top, skip);

        // then
        assertThat(actualResult).isNotNull();
        assertThat(actualResult.getData()).isNotNull();

        PageDto<StsDataDto> pageDto = actualResult.getData();

        assertThat(pageDto.getItems()).hasSize(2);
        assertThat(pageDto.getItems().get(0)).isSameAs(firstDto);
        assertThat(pageDto.getItems().get(1)).isSameAs(secondDto);

        assertThat(pageDto.getPage()).isEqualTo(2);
        assertThat(pageDto.getSize()).isEqualTo(10);
        assertThat(pageDto.getTotalElements()).isEqualTo(35);
        assertThat(pageDto.getTotalPages()).isEqualTo(4);
        assertThat(pageDto.isFirst()).isFalse();
        assertThat(pageDto.isLast()).isFalse();

        verify(stsDataService).getPageByContractUuid(contractUuid, top, skip);
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

        verify(stsDataService).getByUuid(uuid);
        verify(stsDataMapper).toDto(entity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper);
    }
}
```
