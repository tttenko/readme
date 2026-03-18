```java

@ExtendWith(MockitoExtension.class)
class StsDataControllerImplUnitTest {

    @Mock
    private StsDataService stsDataService;

    @Mock
    private StsDataMapper stsDataMapper;

    @Mock
    private StsDataPageMapper stsDataPageMapper;

    @InjectMocks
    private StsDataControllerImpl stsDataController;

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
        verifyNoMoreInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }

    @Test
    void givenContractUuidAndPaging_whenGetStsDataByContractUuid_thenReturnWrappedPageDto() {
        // given
        UUID contractUuid = UUID.randomUUID();
        Integer top = 20;
        Integer skip = 40;

        StsDataEntity firstEntity = new StsDataEntity();
        firstEntity.setUuid(UUID.randomUUID());

        StsDataEntity secondEntity = new StsDataEntity();
        secondEntity.setUuid(UUID.randomUUID());

        Page<StsDataEntity> page = new PageImpl<>(List.of(firstEntity, secondEntity));

        @SuppressWarnings("unchecked")
        PageDto<StsDataDto> pageDto = mock(PageDto.class);

        when(stsDataService.getPageByContractUuid(contractUuid, top, skip)).thenReturn(page);
        when(stsDataPageMapper.toDto(page)).thenReturn(pageDto);

        // when
        ResultObj<PageDto<StsDataDto>> actualResult =
                stsDataController.getStsDataByContractUuid(contractUuid, null, null, top, skip);

        // then
        assertThat(actualResult).isNotNull();
        assertThat(actualResult.getData()).isSameAs(pageDto);

        verify(stsDataService).getPageByContractUuid(contractUuid, top, skip);
        verify(stsDataPageMapper).toDto(page);
        verifyNoMoreInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
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
        verifyNoMoreInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }
}
```
