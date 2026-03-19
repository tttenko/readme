```java
@WebMvcTest(StsDataControllerImpl.class)
@AutoConfigureMockMvc(addFilters = false)
class StsDataControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockitoBean
    private StsDataService stsDataService;

    @MockitoBean
    private StsDataMapper stsDataMapper;

    @Test
    void givenValidRequest_whenCreateStsData_thenReturnOk() throws Exception {
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

        StsDataEntity savedEntity = new StsDataEntity();
        savedEntity.setUuid(uuid);

        StsDataDto dto = new StsDataDto();
        dto.setUuid(uuid);
        dto.setContractUuid(contractUuid);
        dto.setTbCode("1234");
        dto.setVehicleNumber("A123AA777");
        dto.setVehicleBrand("КамАЗ");
        dto.setComment("Тестовая запись");
        dto.setStatusId("01");
        dto.setCreatedBy("system");
        dto.setUpdatedBy("system");
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
                .andExpect(jsonPath("$.count").value(1))
                .andExpect(jsonPath("$.data.uuid").value(uuid.toString()))
                .andExpect(jsonPath("$.data.contractUuid").value(contractUuid.toString()))
                .andExpect(jsonPath("$.data.tbCode").value("1234"))
                .andExpect(jsonPath("$.data.vehicleNumber").value("A123AA777"))
                .andExpect(jsonPath("$.data.vehicleBrand").value("КамАЗ"))
                .andExpect(jsonPath("$.data.comment").value("Тестовая запись"))
                .andExpect(jsonPath("$.data.statusId").value("01"))
                .andExpect(jsonPath("$.data.createdBy").value("system"))
                .andExpect(jsonPath("$.data.updatedBy").value("system"));

        verify(stsDataMapper).toEntity(any(CreateStsDataRequest.class));
        verify(stsDataService).create(mappedEntity);
        verify(stsDataMapper).toDto(savedEntity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper);
    }

    @Test
    void givenInvalidRequest_whenCreateStsData_thenReturnBadRequest() throws Exception {
        // given
        String invalidRequestBody = """
                {
                    "tbCode": "",
                    "vehicleNumber": "A123AA777",
                    "vehicleBrand": "КамАЗ"
                }
                """;

        // when / then
        mockMvc.perform(post("/ui/v1/sts")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidRequestBody))
                .andExpect(status().isBadRequest());

        verifyNoInteractions(stsDataService, stsDataMapper);
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
        dto.setVehicleNumber("A123AA777");
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
                .andExpect(jsonPath("$.count").value(1))
                .andExpect(jsonPath("$.data.uuid").value(uuid.toString()))
                .andExpect(jsonPath("$.data.contractUuid").value(contractUuid.toString()));

        verify(stsDataService).getByUuid(uuid);
        verify(stsDataMapper).toDto(entity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper);
    }

    @Test
    void givenInvalidUuid_whenGetStsDataByUuid_thenReturnBadRequest() throws Exception {
        mockMvc.perform(get("/ui/v1/sts/{uuid}", "not-a-uuid"))
                .andExpect(status().isBadRequest());

        verifyNoInteractions(stsDataService, stsDataMapper);
    }

    @Test
    void givenContractUuidTopSkip_whenGetPage_thenReturnOk() throws Exception {
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

        Page<StsDataEntity> page = new PageImpl<>(
                List.of(firstEntity, secondEntity),
                PageRequest.of(0, 10),
                35
        );

        when(stsDataService.getPageByContractUuid(eq(contractUuid), any(TopSkipOrderFilterParams.class)))
                .thenReturn(page);
        when(stsDataMapper.toDto(firstEntity)).thenReturn(firstDto);
        when(stsDataMapper.toDto(secondEntity)).thenReturn(secondDto);

        // when / then
        mockMvc.perform(get("/ui/v1/contract/{contractUuid}/sts", contractUuid)
                        .param("$top", "10")
                        .param("$skip", "0"))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.count").value(35))
                .andExpect(jsonPath("$.data.length()").value(2))
                .andExpect(jsonPath("$.data[0].uuid").value(firstEntity.getUuid().toString()))
                .andExpect(jsonPath("$.data[1].uuid").value(secondEntity.getUuid().toString()));

        verify(stsDataService).getPageByContractUuid(
                eq(contractUuid),
                argThat(params ->
                        params != null
                                && Integer.valueOf(10).equals(params.getTop())
                                && Integer.valueOf(0).equals(params.getSkip()))
        );
        verify(stsDataMapper).toDto(firstEntity);
        verify(stsDataMapper).toDto(secondEntity);
        verifyNoMoreInteractions(stsDataService, stsDataMapper);
    }
}
```
