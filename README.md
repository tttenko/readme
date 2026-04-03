```java
class StsDataControllerUiIntegrationTest extends StsIntegrationTestBase {

    private static final String UI_V1_PREFIX = "/u1/v1";

    @Autowired
    private StsDataRepository stsDataRepository;

    @Autowired
    private TestHelperService testHelperService;

    @MockitoSpyBean
    private CreateEventOutboxMessageAction createEventOutboxMessageAction;

    @MockitoSpyBean
    private SendTrackerHistoryOutboxMessageAction sendTrackerHistoryOutboxMessageAction;

    @Test
    void toApproveStsDataBySupplier() throws Exception {
        UUID supplierUserId = UUID.randomUUID();
        UUID supplierId = UUID.randomUUID();

        setAuthentication(buildAuthorizedSupplierUser(
                supplierUserId,
                supplierId,
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

        testHelperService.waitTillAllOutboxMessagesProcessed();

        verify(createEventOutboxMessageAction, times(1)).execute(any());
        verify(sendTrackerHistoryOutboxMessageAction, times(1)).execute(any());

        StsDataEntity updatedEntity = stsDataRepository.findByUuidAndDeleted(sts.getUuid(), false)
                .orElseThrow();

        assertEquals(StsStatus.TO_APPROVE_IN, updatedEntity.getStatusId());
    }

    @Test
    void toApproveStsDataByInternalUser() throws Exception {
        UUID supplierUserId = UUID.randomUUID();
        UUID supplierId = UUID.randomUUID();

        setAuthentication(buildAuthorizedSupplierUser(
                supplierUserId,
                supplierId,
                Set.of()
        ));

        StsDataEntity sts = createDraftSts();

        ToApproveStsDataRequest request = new ToApproveStsDataRequest();
        request.setUuids(List.of(sts.getUuid()));

        setAuthentication(buildAuthorizedInternalUser(
                authInternalUser.getUuid(),
                Set.of()
        ));

        mockMvc.perform(patch(UI_V1_PREFIX + "/sts/to_approve")
                        .content(mapper.writeValueAsString(request))
                        .contentType(MediaType.APPLICATION_JSON))
                .andDo(print())
                .andExpect(status().isForbidden());

        verify(createEventOutboxMessageAction, never()).execute(any());
        verify(sendTrackerHistoryOutboxMessageAction, never()).execute(any());

        StsDataEntity notUpdatedEntity = stsDataRepository.findByUuidAndDeleted(sts.getUuid(), false)
                .orElseThrow();

        assertEquals(StsStatus.DRAFT, notUpdatedEntity.getStatusId());
    }

    @Test
    void toApproveStsDataWithoutAuthentication() throws Exception {
        ToApproveStsDataRequest request = new ToApproveStsDataRequest();
        request.setUuids(List.of(UUID.randomUUID()));

        clearAuthentication();

        mockMvc.perform(patch(UI_V1_PREFIX + "/sts/to_approve")
                        .content(mapper.writeValueAsString(request))
                        .contentType(MediaType.APPLICATION_JSON))
                .andDo(print())
                .andExpect(status().isUnauthorized());
    }

    private StsDataEntity createDraftSts() {
        StsDataEntity entity = new StsDataEntity();
        entity.setContractUuid(UUID.randomUUID());
        entity.setTbCode("1234");
        entity.setVehicleNumber("A123AA77");
        entity.setVehicleBrand("Камаз");
        entity.setComment("Тест");
        entity.setStatusId(StsStatus.DRAFT);
        entity.setDeleted(false);

        return stsDataRepository.saveAndFlush(entity);
    }
}
```
