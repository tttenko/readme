```java

 @ExtendWith(MockitoExtension.class)
class AdapterCacheOpsTest {

    @Mock
    BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    SearchRequestProperties properties;

    @Mock
    MeasureUnitMapper measureUnitMapper;

    @Mock
    MaterialTypeMapper materialTypeMapper;

    @Mock
    MaterialMapper materialMapper;

    @InjectMocks
    AdapterCacheOps adapterCacheOps;

    // ---------- getAll* (кэшируемые) ----------

    @Test
    void getAllUoms_whenCalled_thenDelegatesToBackend() {
        // given
        when(properties.getSlugValueForMeasureUnit()).thenReturn("measureUnit");
        when(baseMasterDataRequestService.requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.BOOK))
        ).thenReturn(new GetItemsSearchResponse());

        // when
        List<UomBankDto> result = adapterCacheOps.getAllUoms();

        // then
        assertThat(result).isNotNull();          // сам результат нас тут не волнует
        verify(properties).getSlugValueForMeasureUnit();
        verify(baseMasterDataRequestService).requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.BOOK)
        );
        verifyNoMoreInteractions(baseMasterDataRequestService);
    }

    @Test
    void getAllMaterialTypes_whenCalled_thenDelegatesToBackend() {
        // given
        when(properties.getSlugValueForMaterialType()).thenReturn("materialType");
        when(baseMasterDataRequestService.requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.TMC))
        ).thenReturn(new GetItemsSearchResponse());

        // when
        List<MaterialTypeDto> result = adapterCacheOps.getAllMaterialTypes();

        // then
        assertThat(result).isNotNull();
        verify(properties).getSlugValueForMaterialType();
        verify(baseMasterDataRequestService).requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.TMC)
        );
        verifyNoMoreInteractions(baseMasterDataRequestService);
    }

    @Test
    void getAllMaterials_whenCalled_thenDelegatesToBackend() {
        // given
        when(properties.getSlugValueForMaterial()).thenReturn("material");
        when(baseMasterDataRequestService.requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.TMC))
        ).thenReturn(new GetItemsSearchResponse());

        // when
        List<MaterialDto> result = adapterCacheOps.getAllMaterials();

        // then
        assertThat(result).isNotNull();
        verify(properties).getSlugValueForMaterial();
        verify(baseMasterDataRequestService).requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.TMC)
        );
        verifyNoMoreInteractions(baseMasterDataRequestService);
    }

    // ---------- load* (без кэша) ----------

    @Test
    void loadUomsByCodes_whenCodesPassed_thenUsesMeasureUnitSlugAndBookContext() {
        // given
        List<String> codes = List.of("EA", "KG");
        when(properties.getSlugValueForMeasureUnit()).thenReturn("measureUnit");
        when(baseMasterDataRequestService.requestDataWithAttribute(
                eq("measureUnit"),
                eq(codes),
                eq(SearchRequestProperties.Context.BOOK))
        ).thenReturn(new GetItemsSearchResponse());

        // when
        List<UomBankDto> result = adapterCacheOps.loadUomsByCodes(codes);

        // then
        assertThat(result).isNotNull();
        verify(properties).getSlugValueForMeasureUnit();
        verify(baseMasterDataRequestService).requestDataWithAttribute(
                "measureUnit",
                codes,
                SearchRequestProperties.Context.BOOK
        );
        verifyNoMoreInteractions(baseMasterDataRequestService);
    }

    @Test
    void loadMaterialTypesByIds_whenIdsPassed_thenUsesMaterialTypeSlugAndTmcContext() {
        // given
        List<String> ids = List.of("MT1", "MT2");
        when(properties.getSlugValueForMaterialType()).thenReturn("materialType");
        when(baseMasterDataRequestService.requestDataWithAttribute(
                eq("materialType"),
                eq(ids),
                eq(SearchRequestProperties.Context.TMC))
        ).thenReturn(new GetItemsSearchResponse());

        // when
        List<MaterialTypeDto> result = adapterCacheOps.loadMaterialTypesByIds(ids);

        // then
        assertThat(result).isNotNull();
        verify(properties).getSlugValueForMaterialType();
        verify(baseMasterDataRequestService).requestDataWithAttribute(
                "materialType",
                ids,
                SearchRequestProperties.Context.TMC
        );
        verifyNoMoreInteractions(baseMasterDataRequestService);
    }

    @Test
    void loadMaterialsByCodes_whenCodesPassed_thenUsesMaterialSlugAndTmcContext() {
        // given
        List<String> codes = List.of("M1", "M2");
        when(properties.getSlugValueForMaterial()).thenReturn("material");
        when(baseMasterDataRequestService.requestDataWithAttribute(
                eq("material"),
                eq(codes),
                eq(SearchRequestProperties.Context.TMC))
        ).thenReturn(new GetItemsSearchResponse());

        // when
        List<MaterialDto> result = adapterCacheOps.loadMaterialsByCodes(codes);

        // then
        assertThat(result).isNotNull();
        verify(properties).getSlugValueForMaterial();
        verify(baseMasterDataRequestService).requestDataWithAttribute(
                "material",
                codes,
                SearchRequestProperties.Context.TMC
        );
        verifyNoMoreInteractions(baseMasterDataRequestService);
    }
}
```
