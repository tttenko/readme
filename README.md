```java
@Import(TestConfig.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(properties = {
        "spring.config.location=classpath:/application-test.yml"
})
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@ActiveProfiles("test")
@AutoConfigureMockMvc
public abstract class StsIntegrationTestBase {

    protected static final UUID DEFAULT_SUPPLIER_UUID = UUID.randomUUID();
    protected static final UUID AUTH_INTERNAL_USER_UUID = UUID.randomUUID();
    protected static final UUID AUTH_SUPPLIER_USER_UUID = UUID.randomUUID();

    @Value("${server.servlet.context-path:/control-vehicle}")
    protected String basePath;

    @Autowired
    protected MockMvc mockMvc;

    @Autowired
    protected WebApplicationContext webApplicationContext;

    @Autowired
    protected StsDataRepository stsDataRepository;

    protected final ObjectMapper mapper = JsonMapper.builder()
            .addModule(new JavaTimeModule())
            .build()
            .setSerializationInclusion(JsonInclude.Include.NON_NULL);

    @BeforeAll
    void initBase() {
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
        mockMvc = webAppContextSetup(webApplicationContext).build();
    }

    @BeforeEach
    void setUpBase() {
        SecurityContextHolder.clearContext();
        stsDataRepository.deleteAll();
        mockMvc = webAppContextSetup(webApplicationContext).build();
    }

    protected void setAuthentication(AuthorizedUser principal) {
        UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(principal, null, Set.of());

        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(auth);
        SecurityContextHolder.setContext(context);

        mockMvc = webAppContextSetup(webApplicationContext).build();
    }

    protected void clearAuthentication() {
        SecurityContextHolder.clearContext();
        mockMvc = webAppContextSetup(webApplicationContext).build();
    }

    protected AuthorizedUser buildAuthorizedInternalUser(UUID userId, Set<String> permissions) {
        return new AuthorizedUser(
                Map.of(
                        "EFS_CS_PORTAL_SERVICE_ORDER_MANAGER",
                        new AccessControlRule(
                                permissions,
                                Map.of(
                                        "ABAC_ATTRIBUTE_1", Set.of("ABAC_ATTRIBUTE_VAL_1", "ABAC_ATTRIBUTE_VAL_1_1"),
                                        "ABAC_ATTRIBUTE_2", Set.of("ABAC_ATTRIBUTE_VAL_2", "ABAC_ATTRIBUTE_VAL_2_1")
                                )
                        )
                ),
                new AuthorizationMetadata(
                        Map.of(
                                "USER_FIRST_NAME", "Иван",
                                "USER_MIDDLE_NAME", "Иванович",
                                "USER_LAST_NAME", "Иванов",
                                "USER_POSITION", "Директор"
                        ),
                        Map.of(
                                "USER_PROFILE_UUID", userId.toString(),
                                "USER_ORIGINAL_ID", UUID.randomUUID().toString(),
                                "USER_ORIGIN", UserOrigin.SUDIR.toString()
                        )
                )
        );
    }

    protected AuthorizedUser buildAuthorizedSupplierUser(UUID userId, UUID supplierId, Set<String> permissions) {
        return new AuthorizedUser(
                Map.of(
                        "CSPORTAL_SUPPLIER_ADMIN",
                        new AccessControlRule(
                                permissions,
                                Map.of(
                                        "ABAC_ATTRIBUTE_1", Set.of("ABAC_ATTRIBUTE_VAL_1", "ABAC_ATTRIBUTE_VAL_1_1"),
                                        "ABAC_ATTRIBUTE_2", Set.of("ABAC_ATTRIBUTE_VAL_2", "ABAC_ATTRIBUTE_VAL_2_1"),
                                        "SUPPLIER_MASTER_DATA_ID", Set.of("SUPPLIER_MASTER_DATA_1", "SUPPLIER_MASTER_DATA_2")
                                )
                        )
                ),
                new AuthorizationMetadata(
                        Map.of(
                                "USER_FIRST_NAME", "Иван",
                                "USER_MIDDLE_NAME", "Иванович",
                                "USER_LAST_NAME", "Иванов",
                                "USER_POSITION", "Директор",
                                "SUPPLIER_INN", "5405299839",
                                "SUPPLIER_KPP", "541001001"
                        ),
                        Map.of(
                                "USER_PROFILE_UUID", userId.toString(),
                                "USER_ORIGINAL_ID", UUID.randomUUID().toString(),
                                "USER_ORIGIN", UserOrigin.SBBID.toString(),
                                "SUPPLIER_UUID", supplierId.toString(),
                                "SUPPLIER_USER_EPK_ID", "SomeUserEpkId",
                                "SUPPLIER_EPK_ID", "SomeSupplierEpkId",
                                "USER_LOGIN", "SomeLogin"
                        )
                )
        );
    }

    protected StsDataEntity createDraftSts() {
        StsDataEntity entity = new StsDataEntity();
        entity.setContractUuid(UUID.randomUUID());
        entity.setTbCode("1234");
        entity.setVehicleNumber("A123AA77");
        entity.setVehicleBrand("Камаз");
        entity.setComment("Тестовая запись");
        entity.setStatusId(StsStatus.DRAFT);
        entity.setDeleted(false);

        return stsDataRepository.saveAndFlush(entity);
    }
}

class StsDataControllerUiIntegrationTest extends StsIntegrationTestBase {

    private static final String UI_V1_PREFIX = "/u1/v1";

    @Autowired
    private StsDataRepository stsDataRepository;

    @MockBean
    private StsEventsHistoryOutboxService stsEventsHistoryOutboxService;

    @MockBean
    private StsTrackerHistoryOutboxService stsTrackerHistoryOutboxService;

    @Test
    void toApproveStsDataBySupplier() throws Exception {
        setAuthentication(buildAuthorizedSupplierUser(
                AUTH_SUPPLIER_USER_UUID,
                DEFAULT_SUPPLIER_UUID,
                Set.of()
        ));

        StsDataEntity sts = createDraftSts();

        ToApproveStsDataRequest request = new ToApproveStsDataRequest();
        request.setUuids(List.of(sts.getUuid()));

        mockMvc.perform(patch(UI_V1_PREFIX + "/sts/to_approve")
                        .content(mapper.writeValueAsString(request))
                        .contentType(MediaType.APPLICATION_JSON))
                .andDo(print())
                .andExpect(status().isOk());

        verify(stsEventsHistoryOutboxService, times(1)).sendToApproveEvent(any(), any());
        verify(stsTrackerHistoryOutboxService, times(1)).sendToApproveStatus(any());

        StsDataEntity updatedEntity = stsDataRepository.findByUuidAndDeleted(sts.getUuid(), false)
                .orElseThrow();

        assertEquals(StsStatus.TO_APPROVE_IN, updatedEntity.getStatusId());
    }

    @Test
    void toApproveStsDataByInternalUser() throws Exception {
        setAuthentication(buildAuthorizedInternalUser(
                AUTH_INTERNAL_USER_UUID,
                Set.of()
        ));

        StsDataEntity sts = createDraftSts();

        ToApproveStsDataRequest request = new ToApproveStsDataRequest();
        request.setUuids(List.of(sts.getUuid()));

        mockMvc.perform(patch(UI_V1_PREFIX + "/sts/to_approve")
                        .content(mapper.writeValueAsString(request))
                        .contentType(MediaType.APPLICATION_JSON))
                .andDo(print())
                .andExpect(status().isForbidden());

        verify(stsEventsHistoryOutboxService, never()).sendToApproveEvent(any(), any());
        verify(stsTrackerHistoryOutboxService, never()).sendToApproveStatus(any());

        StsDataEntity entity = stsDataRepository.findByUuidAndDeleted(sts.getUuid(), false)
                .orElseThrow();

        assertEquals(StsStatus.DRAFT, entity.getStatusId());
    }

    @Test
    void toApproveStsDataWithoutAuthentication() throws Exception {
        clearAuthentication();

        ToApproveStsDataRequest request = new ToApproveStsDataRequest();
        request.setUuids(List.of(createDraftSts().getUuid()));

        mockMvc.perform(patch(UI_V1_PREFIX + "/sts/to_approve")
                        .content(mapper.writeValueAsString(request))
                        .contentType(MediaType.APPLICATION_JSON))
                .andDo(print())
                .andExpect(status().isUnauthorized());

        verify(stsEventsHistoryOutboxService, never()).sendToApproveEvent(any(), any());
        verify(stsTrackerHistoryOutboxService, never()).sendToApproveStatus(any());
    }
}
```
