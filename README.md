```java

@WebMvcTest(StsDataControllerImpl.class)
class StsDataControllerWebMvcTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private StsDataService stsDataService;

    @MockBean
    private StsDataMapper stsDataMapper;

    @MockBean
    private StsDataPageMapper stsDataPageMapper;

    @Test
    void givenValidRequest_whenCreateStsData_thenReturnOk() throws Exception {
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
        StsDataEntity savedEntity = new StsDataEntity();
        savedEntity.setUuid(uuid);

        StsDataDto dto = new StsDataDto();
        dto.setUuid(uuid);
        dto.setContractUuid(contractUuid);
        dto.setTbCode("1234");
        dto.setVehicleNumber("А123АА777");
        dto.setVehicleBrand("КамАЗ");
        dto.setComment("Тестовая запись");
        dto.setStatusId("01");
        dto.setCreatedBy("supplier_user");
        dto.setUpdatedBy("supplier_user");
        dto.setDeleted(false);

        when(stsDataMapper.toEntity(any(CreateStsDataRequest.class))).thenReturn(mappedEntity);
        when(stsDataService.create(mappedEntity)).thenReturn(savedEntity);
        when(stsDataMapper.toDto(savedEntity)).thenReturn(dto);

        // when / then
        mockMvc.perform(post("/ui/v1/sts")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.data.uuid").value(uuid.toString()))
                .andExpect(jsonPath("$.data.contractUuid").value(contractUuid.toString()))
                .andExpect(jsonPath("$.data.tbCode").value("1234"))
                .andExpect(jsonPath("$.data.vehicleNumber").value("А123АА777"))
                .andExpect(jsonPath("$.data.vehicleBrand").value("КамАЗ"))
                .andExpect(jsonPath("$.data.comment").value("Тестовая запись"))
                .andExpect(jsonPath("$.data.statusId").value("01"))
                .andExpect(jsonPath("$.data.createdBy").value("supplier_user"));

        verify(stsDataMapper).toEntity(any(CreateStsDataRequest.class));
        verify(stsDataService).create(mappedEntity);
        verify(stsDataMapper).toDto(savedEntity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }

    @Test
    void givenInvalidRequest_whenCreateStsData_thenReturnBadRequest() throws Exception {
        // given
        String invalidRequestBody = """
                {
                  "tbCode": "",
                  "vehicleNumber": "А123АА777",
                  "vehicleBrand": "КамАЗ",
                  "statusId": "01",
                  "createdBy": "supplier_user"
                }
                """;

        // when / then
        mockMvc.perform(post("/ui/v1/sts")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidRequestBody))
                .andExpect(status().isBadRequest());

        verifyNoInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }

    @Test
    void givenValidUuid_whenGetStsDataByUuid_thenReturnOk() throws Exception {
        // given
        UUID uuid = UUID.randomUUID();
        UUID contractUuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(uuid);

        StsDataDto dto = new StsDataDto();
        dto.setUuid(uuid);
        dto.setContractUuid(contractUuid);
        dto.setTbCode("1234");
        dto.setVehicleNumber("А123АА777");
        dto.setVehicleBrand("КамАЗ");
        dto.setComment("Карточка");
        dto.setStatusId("01");
        dto.setCreatedBy("user_1");
        dto.setUpdatedBy("user_1");
        dto.setDeleted(false);

        when(stsDataService.getByUuid(uuid)).thenReturn(entity);
        when(stsDataMapper.toDto(entity)).thenReturn(dto);

        // when / then
        mockMvc.perform(get("/ui/v1/sts/{uuid}", uuid))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.data.uuid").value(uuid.toString()))
                .andExpect(jsonPath("$.data.contractUuid").value(contractUuid.toString()));

        verify(stsDataService).getByUuid(uuid);
        verify(stsDataMapper).toDto(entity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }

    @Test
    void givenInvalidUuid_whenGetStsDataByUuid_thenReturnBadRequest() throws Exception {
        mockMvc.perform(get("/ui/v1/sts/{uuid}", "not-a-uuid"))
                .andExpect(status().isBadRequest());

        verifyNoInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }

    @Test
    void givenContractUuidTopSkip_whenGetPage_thenReturnOk() throws Exception {
        // given
        UUID contractUuid = UUID.randomUUID();

        Page<StsDataEntity> page = new PageImpl<>(List.of(new StsDataEntity(), new StsDataEntity()));

        @SuppressWarnings("unchecked")
        PageDto<StsDataDto> pageDto = mock(PageDto.class);

        when(stsDataService.getPageByContractUuid(contractUuid, 10, 0)).thenReturn(page);
        when(stsDataPageMapper.toDto(page)).thenReturn(pageDto);

        // when / then
        mockMvc.perform(get("/ui/v1/contract/{contractUuid}/sts", contractUuid)
                        .param("$top", "10")
                        .param("$skip", "0"))
                .andExpect(status().isOk());

        verify(stsDataService).getPageByContractUuid(contractUuid, 10, 0);
        verify(stsDataPageMapper).toDto(page);
        verifyNoMoreInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }
}

@WebMvcTest(StsDataControllerImpl.class)
class StsDataControllerWebMvcTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private StsDataService stsDataService;

    @MockBean
    private StsDataMapper stsDataMapper;

    @MockBean
    private StsDataPageMapper stsDataPageMapper;

    @Test
    void givenValidRequest_whenCreateStsData_thenReturnOk() throws Exception {
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
        StsDataEntity savedEntity = new StsDataEntity();
        savedEntity.setUuid(uuid);

        StsDataDto dto = new StsDataDto();
        dto.setUuid(uuid);
        dto.setContractUuid(contractUuid);
        dto.setTbCode("1234");
        dto.setVehicleNumber("А123АА777");
        dto.setVehicleBrand("КамАЗ");
        dto.setComment("Тестовая запись");
        dto.setStatusId("01");
        dto.setCreatedBy("supplier_user");
        dto.setUpdatedBy("supplier_user");
        dto.setDeleted(false);

        when(stsDataMapper.toEntity(any(CreateStsDataRequest.class))).thenReturn(mappedEntity);
        when(stsDataService.create(mappedEntity)).thenReturn(savedEntity);
        when(stsDataMapper.toDto(savedEntity)).thenReturn(dto);

        // when / then
        mockMvc.perform(post("/ui/v1/sts")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.data.uuid").value(uuid.toString()))
                .andExpect(jsonPath("$.data.contractUuid").value(contractUuid.toString()))
                .andExpect(jsonPath("$.data.tbCode").value("1234"))
                .andExpect(jsonPath("$.data.vehicleNumber").value("А123АА777"))
                .andExpect(jsonPath("$.data.vehicleBrand").value("КамАЗ"))
                .andExpect(jsonPath("$.data.comment").value("Тестовая запись"))
                .andExpect(jsonPath("$.data.statusId").value("01"))
                .andExpect(jsonPath("$.data.createdBy").value("supplier_user"));

        verify(stsDataMapper).toEntity(any(CreateStsDataRequest.class));
        verify(stsDataService).create(mappedEntity);
        verify(stsDataMapper).toDto(savedEntity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }

    @Test
    void givenInvalidRequest_whenCreateStsData_thenReturnBadRequest() throws Exception {
        // given
        String invalidRequestBody = """
                {
                  "tbCode": "",
                  "vehicleNumber": "А123АА777",
                  "vehicleBrand": "КамАЗ",
                  "statusId": "01",
                  "createdBy": "supplier_user"
                }
                """;

        // when / then
        mockMvc.perform(post("/ui/v1/sts")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidRequestBody))
                .andExpect(status().isBadRequest());

        verifyNoInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }

    @Test
    void givenValidUuid_whenGetStsDataByUuid_thenReturnOk() throws Exception {
        // given
        UUID uuid = UUID.randomUUID();
        UUID contractUuid = UUID.randomUUID();

        StsDataEntity entity = new StsDataEntity();
        entity.setUuid(uuid);

        StsDataDto dto = new StsDataDto();
        dto.setUuid(uuid);
        dto.setContractUuid(contractUuid);
        dto.setTbCode("1234");
        dto.setVehicleNumber("А123АА777");
        dto.setVehicleBrand("КамАЗ");
        dto.setComment("Карточка");
        dto.setStatusId("01");
        dto.setCreatedBy("user_1");
        dto.setUpdatedBy("user_1");
        dto.setDeleted(false);

        when(stsDataService.getByUuid(uuid)).thenReturn(entity);
        when(stsDataMapper.toDto(entity)).thenReturn(dto);

        // when / then
        mockMvc.perform(get("/ui/v1/sts/{uuid}", uuid))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.data.uuid").value(uuid.toString()))
                .andExpect(jsonPath("$.data.contractUuid").value(contractUuid.toString()));

        verify(stsDataService).getByUuid(uuid);
        verify(stsDataMapper).toDto(entity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }

    @Test
    void givenInvalidUuid_whenGetStsDataByUuid_thenReturnBadRequest() throws Exception {
        mockMvc.perform(get("/ui/v1/sts/{uuid}", "not-a-uuid"))
                .andExpect(status().isBadRequest());

        verifyNoInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }

    @Test
    void givenContractUuidTopSkip_whenGetPage_thenReturnOk() throws Exception {
        // given
        UUID contractUuid = UUID.randomUUID();

        Page<StsDataEntity> page = new PageImpl<>(List.of(new StsDataEntity(), new StsDataEntity()));

        @SuppressWarnings("unchecked")
        PageDto<StsDataDto> pageDto = mock(PageDto.class);

        when(stsDataService.getPageByContractUuid(contractUuid, 10, 0)).thenReturn(page);
        when(stsDataPageMapper.toDto(page)).thenReturn(pageDto);

        // when / then
        mockMvc.perform(get("/ui/v1/contract/{contractUuid}/sts", contractUuid)
                        .param("$top", "10")
                        .param("$skip", "0"))
                .andExpect(status().isOk());

        verify(stsDataService).getPageByContractUuid(contractUuid, 10, 0);
        verify(stsDataPageMapper).toDto(page);
        verifyNoMoreInteractions(stsDataService, stsDataMapper, stsDataPageMapper);
    }
}
```
