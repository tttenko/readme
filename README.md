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
        ZonedDateTime now = ZonedDateTime.now();

        CreateStsDataRequest request = new CreateStsDataRequest();
        request.setContractUuid(contractUuid);
        request.setTbId("1234");
        request.setVehicleNumber("A123AA777");
        request.setVehicleBrand("КамАЗ");
        request.setComment("Тестовая запись");
        request.setStatusId("01");
        request.setCreatedBy("supplier_user");

        StsDataDto responseDto = new StsDataDto();
        responseDto.setUuid(uuid);
        responseDto.setContractUuid(contractUuid);
        responseDto.setTbId("1234");
        responseDto.setVehicleNumber("A123AA777");
        responseDto.setVehicleBrand("КамАЗ");
        responseDto.setComment("Тестовая запись");
        responseDto.setStatusId("01");
        responseDto.setCreatedBy("supplier_user");
        responseDto.setUpdatedBy("supplier_user");
        responseDto.setCreatedAt(now);
        responseDto.setUpdatedAt(now);
        responseDto.setDeleted(false);

        when(stsDataService.create(any(CreateStsDataRequest.class))).thenReturn(responseDto);

        // when / then
        mockMvc.perform(post("/api/v1/sts")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))

                .andExpect(jsonPath("$.count").value(1))
                .andExpect(jsonPath("$.data", hasSize(1)))

                .andExpect(jsonPath("$.data[0].uuid").value(uuid.toString()))
                .andExpect(jsonPath("$.data[0].contractUuid").value(contractUuid.toString()))
                .andExpect(jsonPath("$.data[0].tbId").value("1234"))
                .andExpect(jsonPath("$.data[0].vehicleNumber").value("A123AA777"))
                .andExpect(jsonPath("$.data[0].vehicleBrand").value("КамАЗ"))
                .andExpect(jsonPath("$.data[0].comment").value("Тестовая запись"))
                .andExpect(jsonPath("$.data[0].statusId").value("01"))
                .andExpect(jsonPath("$.data[0].createdBy").value("supplier_user"))
                .andExpect(jsonPath("$.data[0].updatedBy").value("supplier_user"))
                .andExpect(jsonPath("$.data[0].deleted").value(false))
                .andExpect(jsonPath("$.data[0].createdAt").exists())
                .andExpect(jsonPath("$.data[0].updatedAt").exists())

                .andExpect(jsonPath("$.messages", hasSize(1)))
                .andExpect(jsonPath("$.messages[0].message").value("Поиск выполнен"))
                .andExpect(jsonPath("$.messages[0].description").value("Найдено записей: 1"));

        verify(stsDataService).create(any(CreateStsDataRequest.class));
        verifyNoMoreInteractions(stsDataService);
    }

    @Test
    void givenInvalidRequest_whenCreateStsData_thenReturnBadRequest() throws Exception {
        // given
        String invalidRequestBody = """
                {
                  "tbId": "",
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
        ZonedDateTime now = ZonedDateTime.now();

        StsDataDto firstDto = new StsDataDto();
        firstDto.setUuid(UUID.randomUUID());
        firstDto.setContractUuid(contractUuid);
        firstDto.setTbId("1234");
        firstDto.setVehicleNumber("A123AA777");
        firstDto.setVehicleBrand("КамАЗ");
        firstDto.setComment("Первая запись");
        firstDto.setStatusId("01");
        firstDto.setCreatedBy("user_1");
        firstDto.setUpdatedBy("user_1");
        firstDto.setCreatedAt(now);
        firstDto.setUpdatedAt(now);
        firstDto.setDeleted(false);

        StsDataDto secondDto = new StsDataDto();
        secondDto.setUuid(UUID.randomUUID());
        secondDto.setContractUuid(contractUuid);
        secondDto.setTbId("5678");
        secondDto.setVehicleNumber("B456BB777");
        secondDto.setVehicleBrand("МАЗ");
        secondDto.setComment("Вторая запись");
        secondDto.setStatusId("02");
        secondDto.setCreatedBy("user_2");
        secondDto.setUpdatedBy("user_2");
        secondDto.setCreatedAt(now);
        secondDto.setUpdatedAt(now);
        secondDto.setDeleted(false);

        when(stsDataService.getByContractUuid(contractUuid)).thenReturn(List.of(firstDto, secondDto));

        // when / then
        mockMvc.perform(get("/api/v1/sts")
                        .param("contractUuid", contractUuid.toString()))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))

                .andExpect(jsonPath("$.count").value(2))
                .andExpect(jsonPath("$.data", hasSize(2)))

                .andExpect(jsonPath("$.data[0].uuid").value(firstDto.getUuid().toString()))
                .andExpect(jsonPath("$.data[0].contractUuid").value(contractUuid.toString()))
                .andExpect(jsonPath("$.data[0].tbId").value("1234"))
                .andExpect(jsonPath("$.data[0].vehicleNumber").value("A123AA777"))
                .andExpect(jsonPath("$.data[0].vehicleBrand").value("КамАЗ"))
                .andExpect(jsonPath("$.data[0].comment").value("Первая запись"))
                .andExpect(jsonPath("$.data[0].statusId").value("01"))
                .andExpect(jsonPath("$.data[0].createdBy").value("user_1"))
                .andExpect(jsonPath("$.data[0].updatedBy").value("user_1"))
                .andExpect(jsonPath("$.data[0].deleted").value(false))

                .andExpect(jsonPath("$.data[1].uuid").value(secondDto.getUuid().toString()))
                .andExpect(jsonPath("$.data[1].contractUuid").value(contractUuid.toString()))
                .andExpect(jsonPath("$.data[1].tbId").value("5678"))
                .andExpect(jsonPath("$.data[1].vehicleNumber").value("B456BB777"))
                .andExpect(jsonPath("$.data[1].vehicleBrand").value("МАЗ"))
                .andExpect(jsonPath("$.data[1].comment").value("Вторая запись"))
                .andExpect(jsonPath("$.data[1].statusId").value("02"))
                .andExpect(jsonPath("$.data[1].createdBy").value("user_2"))
                .andExpect(jsonPath("$.data[1].updatedBy").value("user_2"))
                .andExpect(jsonPath("$.data[1].deleted").value(false))

                .andExpect(jsonPath("$.messages", hasSize(1)))
                .andExpect(jsonPath("$.messages[0].message").value("Поиск выполнен"))
                .andExpect(jsonPath("$.messages[0].description").value("Найдено записей: 2"));

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
        ZonedDateTime now = ZonedDateTime.now();

        StsDataDto dto = new StsDataDto();
        dto.setUuid(uuid);
        dto.setContractUuid(contractUuid);
        dto.setTbId("1234");
        dto.setVehicleNumber("A123AA777");
        dto.setVehicleBrand("КамАЗ");
        dto.setComment("Карточка записи");
        dto.setStatusId("01");
        dto.setCreatedBy("user_1");
        dto.setUpdatedBy("user_1");
        dto.setCreatedAt(now);
        dto.setUpdatedAt(now);
        dto.setDeleted(false);

        when(stsDataService.getByUuid(uuid)).thenReturn(dto);

        // when / then
        mockMvc.perform(get("/api/v1/sts/{uuid}", uuid))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))

                .andExpect(jsonPath("$.count").value(1))

                .andExpect(jsonPath("$.data.uuid").value(uuid.toString()))
                .andExpect(jsonPath("$.data.contractUuid").value(contractUuid.toString()))
                .andExpect(jsonPath("$.data.tbId").value("1234"))
                .andExpect(jsonPath("$.data.vehicleNumber").value("A123AA777"))
                .andExpect(jsonPath("$.data.vehicleBrand").value("КамАЗ"))
                .andExpect(jsonPath("$.data.comment").value("Карточка записи"))
                .andExpect(jsonPath("$.data.statusId").value("01"))
                .andExpect(jsonPath("$.data.createdBy").value("user_1"))
                .andExpect(jsonPath("$.data.updatedBy").value("user_1"))
                .andExpect(jsonPath("$.data.deleted").value(false))
                .andExpect(jsonPath("$.data.createdAt").exists())
                .andExpect(jsonPath("$.data.updatedAt").exists())

                .andExpect(jsonPath("$.messages", hasSize(1)))
                .andExpect(jsonPath("$.messages[0].message").value("Поиск выполнен"))
                .andExpect(jsonPath("$.messages[0].description").value("Объект успешно получен"));

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
}
```
