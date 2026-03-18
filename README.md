```java

@ExtendWith(MockitoExtension.class)
class StsDataControllerImplTest {

    private MockMvc mockMvc;
    private ObjectMapper objectMapper;

    @Mock
    private StsDataService stsDataService;

    @InjectMocks
    private StsDataControllerImpl stsDataController;

    @BeforeEach
    void setUp() {
        objectMapper = new ObjectMapper().findAndRegisterModules();

        LocalValidatorFactoryBean validator = new LocalValidatorFactoryBean();
        validator.afterPropertiesSet();

        mockMvc = MockMvcBuilders.standaloneSetup(stsDataController)
                .setValidator(validator)
                .setMessageConverters(new MappingJackson2HttpMessageConverter(objectMapper))
                .build();
    }

    @Test
    void givenValidRequest_whenCreateStsData_thenReturnOkAndWrappedCreatedItem() throws Exception {
        // given
        UUID uuid = UUID.randomUUID();
        UUID contractUuid = UUID.randomUUID();

        CreateStsDataRequest request = buildCreateRequest(contractUuid);

        StsDataDto responseDto = buildStsDataDto(
                uuid,
                contractUuid,
                "1234",
                "A123AA777",
                "КамАЗ",
                "Тестовая запись",
                "01",
                "supplier_user"
        );

        when(stsDataService.create(any(CreateStsDataRequest.class))).thenReturn(responseDto);

        // when
        ResultActions result = mockMvc.perform(post("/api/v1/sts")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)));

        // then
        assertOkJsonResponse(result);
        assertListResponseMeta(result, 1);
        assertStsDataItem(result, "$.data[0]", responseDto);
        assertSuccessMessage(result, "Найдено записей: 1");

        verify(stsDataService).create(any(CreateStsDataRequest.class));
        verifyNoMoreInteractions(stsDataService);
    }

    @Test
    void givenInvalidRequest_whenCreateStsData_thenReturnBadRequest() throws Exception {
        // given
        String invalidRequestBody = """
                {
                  "tbCode": "",
                  "vehicleNumber": "A123AA777",
                  "vehicleBrand": "КамАЗ",
                  "statusId": "01",
                  "createdBy": "supplier_user"
                }
                """;

        // when / then
        mockMvc.perform(post("/api/v1/sts")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidRequestBody))
                .andExpect(status().isBadRequest());

        verifyNoInteractions(stsDataService);
    }

    @Test
    void givenContractUuid_whenGetStsDataByContractUuid_thenReturnOkAndWrappedItemList() throws Exception {
        // given
        UUID contractUuid = UUID.randomUUID();

        StsDataDto firstDto = buildStsDataDto(
                UUID.randomUUID(),
                contractUuid,
                "1234",
                "A123AA777",
                "КамАЗ",
                "Первая запись",
                "01",
                "user_1"
        );

        StsDataDto secondDto = buildStsDataDto(
                UUID.randomUUID(),
                contractUuid,
                "5678",
                "B456BB777",
                "МАЗ",
                "Вторая запись",
                "02",
                "user_2"
        );

        when(stsDataService.getByContractUuid(contractUuid)).thenReturn(List.of(firstDto, secondDto));

        // when
        ResultActions result = mockMvc.perform(get("/api/v1/sts")
                .param("contractUuid", contractUuid.toString()));

        // then
        assertOkJsonResponse(result);
        assertListResponseMeta(result, 2);
        assertStsDataItem(result, "$.data[0]", firstDto);
        assertStsDataItem(result, "$.data[1]", secondDto);
        assertSuccessMessage(result, "Найдено записей: 2");

        verify(stsDataService).getByContractUuid(contractUuid);
        verifyNoMoreInteractions(stsDataService);
    }

    @Test
    void givenMissingContractUuid_whenGetStsDataByContractUuid_thenReturnBadRequest() throws Exception {
        // when / then
        mockMvc.perform(get("/api/v1/sts"))
                .andExpect(status().isBadRequest());

        verifyNoInteractions(stsDataService);
    }

    @Test
    void givenValidUuid_whenGetStsDataByUuid_thenReturnOkAndWrappedItem() throws Exception {
        // given
        UUID uuid = UUID.randomUUID();
        UUID contractUuid = UUID.randomUUID();

        StsDataDto dto = buildStsDataDto(
                uuid,
                contractUuid,
                "1234",
                "A123AA777",
                "КамАЗ",
                "Карточка записи",
                "01",
                "user_1"
        );

        when(stsDataService.getByUuid(uuid)).thenReturn(dto);

        // when
        ResultActions result = mockMvc.perform(get("/api/v1/sts/{uuid}", uuid));

        // then
        assertOkJsonResponse(result);
        assertItemResponseMeta(result);
        assertStsDataItem(result, "$.data", dto);
        assertItemSuccessMessage(result);

        verify(stsDataService).getByUuid(uuid);
        verifyNoMoreInteractions(stsDataService);
    }

    @Test
    void givenInvalidUuid_whenGetStsDataByUuid_thenReturnBadRequest() throws Exception {
        // when / then
        mockMvc.perform(get("/api/v1/sts/{uuid}", "not-a-uuid"))
                .andExpect(status().isBadRequest());

        verifyNoInteractions(stsDataService);
    }

    private CreateStsDataRequest buildCreateRequest(UUID contractUuid) {
        CreateStsDataRequest request = new CreateStsDataRequest();
        request.setContractUuid(contractUuid);
        request.setTbCode("1234");
        request.setVehicleNumber("A123AA777");
        request.setVehicleBrand("КамАЗ");
        request.setComment("Тестовая запись");
        request.setStatusId("01");
        request.setCreatedBy("supplier_user");
        return request;
    }

    private StsDataDto buildStsDataDto(UUID uuid,
                                       UUID contractUuid,
                                       String tbCode,
                                       String vehicleNumber,
                                       String vehicleBrand,
                                       String comment,
                                       String statusId,
                                       String createdBy) {
        ZonedDateTime now = ZonedDateTime.now();

        StsDataDto dto = new StsDataDto();
        dto.setUuid(uuid);
        dto.setContractUuid(contractUuid);
        dto.setTbCode(tbCode);
        dto.setVehicleNumber(vehicleNumber);
        dto.setVehicleBrand(vehicleBrand);
        dto.setComment(comment);
        dto.setStatusId(statusId);
        dto.setCreatedBy(createdBy);
        dto.setUpdateBy(createdBy);
        dto.setCreatedAt(now);
        dto.setUpdateAt(now);
        dto.setDeleted(false);
        return dto;
    }

    private void assertOkJsonResponse(ResultActions result) throws Exception {
        result.andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON));
    }

    private void assertListResponseMeta(ResultActions result, int expectedCount) throws Exception {
        result.andExpect(jsonPath("$.count").value(expectedCount))
                .andExpect(jsonPath("$.data", hasSize(expectedCount)))
                .andExpect(jsonPath("$.messages", hasSize(1)))
                .andExpect(jsonPath("$.messages[0].message").value("Поиск выполнен"));
    }

    private void assertItemResponseMeta(ResultActions result) throws Exception {
        result.andExpect(jsonPath("$.count").value(1))
                .andExpect(jsonPath("$.messages", hasSize(1)))
                .andExpect(jsonPath("$.messages[0].message").value("Поиск выполнен"));
    }

    private void assertSuccessMessage(ResultActions result, String expectedDescription) throws Exception {
        result.andExpect(jsonPath("$.messages[0].description").value(expectedDescription));
    }

    private void assertItemSuccessMessage(ResultActions result) throws Exception {
        result.andExpect(jsonPath("$.messages[0].description").value("Объект успешно получен"));
    }

    private void assertStsDataItem(ResultActions result, String pathPrefix, StsDataDto dto) throws Exception {
        result.andExpect(jsonPath(pathPrefix + ".uuid").value(dto.getUuid().toString()))
                .andExpect(jsonPath(pathPrefix + ".contractUuid").value(dto.getContractUuid().toString()))
                .andExpect(jsonPath(pathPrefix + ".tbCode").value(dto.getTbCode()))
                .andExpect(jsonPath(pathPrefix + ".vehicleNumber").value(dto.getVehicleNumber()))
                .andExpect(jsonPath(pathPrefix + ".vehicleBrand").value(dto.getVehicleBrand()))
                .andExpect(jsonPath(pathPrefix + ".comment").value(dto.getComment()))
                .andExpect(jsonPath(pathPrefix + ".statusId").value(dto.getStatusId()))
                .andExpect(jsonPath(pathPrefix + ".createdBy").value(dto.getCreatedBy()))
                .andExpect(jsonPath(pathPrefix + ".updateBy").value(dto.getUpdateBy()))
                .andExpect(jsonPath(pathPrefix + ".deleted").value(dto.isDeleted()))
                .andExpect(jsonPath(pathPrefix + ".createdAt").exists())
                .andExpect(jsonPath(pathPrefix + ".updateAt").exists());
    }
}
```
