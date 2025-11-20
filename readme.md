```java

 @ExtendWith(MockitoExtension.class)
class AdapterCacheOpsTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SearchRequestProperties properties;

    @Mock
    private MeasureUnitMapper measureUnitMapper;

    @Mock
    private MaterialTypeMapper materialTypeMapper;

    @Mock
    private MaterialMapper materialMapper;

    @InjectMocks
    private AdapterCacheOps adapterCacheOps;

    @Test
    void getAllUoms_whenBackendReturnsData_thenMapsResult() {
        // given
        when(properties.getSlugValueForMeasureUnit()).thenReturn("measureUnit");

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.BOOK))
        ).thenReturn(resp);

        List<UomBankDto> mapped = List.of(
                UomBankDto.builder().uomCode("EA").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, measureUnitMapper)
            ).thenReturn(mapped);

            // when
            List<UomBankDto> result = adapterCacheOps.getAllUoms();

            // then
            assertThat(result).containsExactlyElementsOf(mapped);

            verify(properties).getSlugValueForMeasureUnit();
            verify(baseMasterDataRequestService).requestData(
                    any(ItemsSearchCriteriaRequest.class),
                    eq(SearchRequestProperties.Context.BOOK)
            );
            statics.verify(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, measureUnitMapper)
            );
        }
    }

    @Test
    void getAllMaterialTypes_whenBackendReturnsData_thenMapsResult() {
        // given
        when(properties.getSlugValueForMaterialType()).thenReturn("materialType");

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.TMC))
        ).thenReturn(resp);

        List<MaterialTypeDto> mapped = List.of(
                MaterialTypeDto.builder().typeId("TYPE1").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, materialTypeMapper)
            ).thenReturn(mapped);

            // when
            List<MaterialTypeDto> result = adapterCacheOps.getAllMaterialTypes();

            // then
            assertThat(result).containsExactlyElementsOf(mapped);

            verify(properties).getSlugValueForMaterialType();
            verify(baseMasterDataRequestService).requestData(
                    any(ItemsSearchCriteriaRequest.class),
                    eq(SearchRequestProperties.Context.TMC)
            );
            statics.verify(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, materialTypeMapper)
            );
        }
    }

    @Test
    void getAllMaterials_whenBackendReturnsData_thenMapsResult() {
        // given
        when(properties.getSlugValueForMaterial()).thenReturn("material");

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.TMC))
        ).thenReturn(resp);

        List<MaterialDto> mapped = List.of(
                MaterialDto.builder().materialCode("M1").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, materialMapper)
            ).thenReturn(mapped);

            // when
            List<MaterialDto> result = adapterCacheOps.getAllMaterials();

            // then
            assertThat(result).containsExactlyElementsOf(mapped);

            verify(properties).getSlugValueForMaterial();
            verify(baseMasterDataRequestService).requestData(
                    any(ItemsSearchCriteriaRequest.class),
                    eq(SearchRequestProperties.Context.TMC)
            );
            statics.verify(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, materialMapper)
            );
        }
    }
}
```
