```java
///**
// * Unit-тесты для {@link LoaderUomByCode}.
// * <p>
// * Проверяет базовую конфигурацию лоадера (имя кэша, тип элемента, ключ),
// * а также то, что при загрузке по ключам он корректно формирует запрос
// * к {@link BaseMasterDataRequestService} и не гасит исключения бэкенда.
// */
@ExtendWith(MockitoExtension.class)
class LoaderUomByCodeTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SearchRequestProperties properties;

    @Mock
    private MeasureUnitMapper measureUnitMapper;

    @InjectMocks
    private LoaderUomByCode loader;

    @Test
    void cacheName_returnsUomByCodeConstant() {
        assertEquals(AdapterService2.UOM_BY_CODE, loader.cacheName());
    }

    @Test
    void elementType_returnsUomBankDtoClass() {
        assertEquals(UomBankDto.class, loader.elementType());
    }

    @Test
    void extractKey_returnsUomCodeFromDto() {
        UomBankDto dto = UomBankDto.builder()
                .uomCode("EA")
                .build();

        assertEquals("EA", loader.extractKey(dto));
    }

    @Test
    void fetchByKeys_givenEmptyKeys_returnsEmptyAndSkipsBackend() {
        // when
        List<UomBankDto> result = loader.fetchByKeys(List.of());

        // then
        assertThat(result).isEmpty();
        verifyNoInteractions(baseMasterDataRequestService, properties, measureUnitMapper);
    }

    @Test
    void fetchByKeys_givenKeys_delegatesToBackendWithSlugAndContext_andPropagatesException() {
        // given
        List<String> keys = List.of("EA", "KG");
        when(properties.getSlugValueForMeasureUnit()).thenReturn("measureUnit");

        RuntimeException backendError = new RuntimeException("boom");
        doThrow(backendError).when(baseMasterDataRequestService)
                .requestDataWithAttribute(anyString(), anyList(), any());

        // when
        RuntimeException thrown = assertThrows(RuntimeException.class,
                () -> loader.fetchByKeys(keys));

        // then
        assertSame(backendError, thrown);

        verify(properties).getSlugValueForMeasureUnit();
        verify(baseMasterDataRequestService).requestDataWithAttribute(
                eq("measureUnit"),
                eq(keys),
                eq(SearchRequestProperties.Context.BOOK)
        );
        verifyNoMoreInteractions(baseMasterDataRequestService);
    }
}


///**
// * Unit-тесты для {@link LoaderMaterialTypeById}.
// * <p>
// * Проверяет имя кэша, тип элемента и ключ для DTO,
// * а также корректную делегацию вызова в {@link BaseMasterDataRequestService}.
// */
@ExtendWith(MockitoExtension.class)
class LoaderMaterialTypeByIdTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SearchRequestProperties properties;

    @Mock
    private MaterialTypeMapper materialTypeMapper;

    @InjectMocks
    private LoaderMaterialTypeById loader;

    @Test
    void cacheName_returnsMaterialTypeByIdConstant() {
        assertEquals(AdapterService2.MATERIAL_TYPE_BY_ID, loader.cacheName());
    }

    @Test
    void elementType_returnsMaterialTypeDtoClass() {
        assertEquals(MaterialTypeDto.class, loader.elementType());
    }

    @Test
    void extractKey_returnsTypeIdFromDto() {
        MaterialTypeDto dto = MaterialTypeDto.builder()
                .typeId("TYPE1")
                .build();

        assertEquals("TYPE1", loader.extractKey(dto));
    }

    @Test
    void fetchByKeys_givenEmptyIds_returnsEmptyAndSkipsBackend() {
        // when
        List<MaterialTypeDto> result = loader.fetchByKeys(List.of());

        // then
        assertThat(result).isEmpty();
        verifyNoInteractions(baseMasterDataRequestService, properties, materialTypeMapper);
    }

    @Test
    void fetchByKeys_givenIds_delegatesToBackendWithSlugAndContext_andPropagatesException() {
        // given
        List<String> ids = List.of("MT1", "MT2");
        when(properties.getSlugValueForMaterialType()).thenReturn("materialType");

        RuntimeException backendError = new RuntimeException("boom");
        doThrow(backendError).when(baseMasterDataRequestService)
                .requestDataWithAttribute(anyString(), anyList(), any());

        // when
        RuntimeException thrown = assertThrows(RuntimeException.class,
                () -> loader.fetchByKeys(ids));

        // then
        assertSame(backendError, thrown);

        verify(properties).getSlugValueForMaterialType();
        verify(baseMasterDataRequestService).requestDataWithAttribute(
                eq("materialType"),
                eq(ids),
                eq(SearchRequestProperties.Context.TMC)
        );
        verifyNoMoreInteractions(baseMasterDataRequestService);
    }
}


///**
// * Unit-тесты для {@link LoaderMaterialByCode}.
// * <p>
// * Проверяет конфигурацию лоадера (имя кэша, тип элемента, ключ)
// * и делегацию вызова в {@link BaseMasterDataRequestService} при загрузке по кодам.
// */
@ExtendWith(MockitoExtension.class)
class LoaderMaterialByCodeTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SearchRequestProperties properties;

    @Mock
    private MaterialMapper materialMapper;

    @InjectMocks
    private LoaderMaterialByCode loader;

    @Test
    void cacheName_returnsMaterialByCodeConstant() {
        assertEquals(AdapterService2.MATERIAL_BY_CODE, loader.cacheName());
    }

    @Test
    void elementType_returnsMaterialDtoClass() {
        assertEquals(MaterialDto.class, loader.elementType());
    }

    @Test
    void extractKey_returnsMaterialCodeFromDto() {
        MaterialDto dto = MaterialDto.builder()
                .materialCode("M1")
                .build();

        assertEquals("M1", loader.extractKey(dto));
    }

    @Test
    void fetchByKeys_givenEmptyCodes_returnsEmptyAndSkipsBackend() {
        // when
        List<MaterialDto> result = loader.fetchByKeys(List.of());

        // then
        assertThat(result).isEmpty();
        verifyNoInteractions(baseMasterDataRequestService, properties, materialMapper);
    }

    @Test
    void fetchByKeys_givenCodes_delegatesToBackendWithSlugAndContext_andPropagatesException() {
        // given
        List<String> codes = List.of("M1", "M2");
        when(properties.getSlugValueForMaterial()).thenReturn("material");

        RuntimeException backendError = new RuntimeException("boom");
        doThrow(backendError).when(baseMasterDataRequestService)
                .requestDataWithAttribute(anyString(), anyList(), any());

        // when
        RuntimeException thrown = assertThrows(RuntimeException.class,
                () -> loader.fetchByKeys(codes));

        // then
        assertSame(backendError, thrown);

        verify(properties).getSlugValueForMaterial();
        verify(baseMasterDataRequestService).requestDataWithAttribute(
                eq("material"),
                eq(codes),
                eq(SearchRequestProperties.Context.TMC)
        );
        verifyNoMoreInteractions(baseMasterDataRequestService);
    }
}


///**
// * Unit-тесты для {@link AdapterService2}.
// * <p>
// * Проверяет, что сервис правильно выбирает источник данных:
// * <ul>
// *     <li>если список кодов / идентификаторов пустой или {@code null},
// *     используются методы {@link AdapterCacheOps} для загрузки «всего»;</li>
// *     <li>если список непустой — запрос уходит в {@link CacheGetOrLoadService}.</li>
// * </ul>
// */
@ExtendWith(MockitoExtension.class)
class AdapterService2Test {

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    @Mock
    private AdapterCacheOps adapterCacheOps;

    @InjectMocks
    private AdapterService2 adapterService2;

    // ---------- UOM ----------

    @Test
    void getUom_whenCodesNull_thenUsesAllFromCacheOps() {
        // given
        List<UomBankDto> all = List.of(
                UomBankDto.builder().uomCode("EA").build()
        );
        when(adapterCacheOps.getAllUoms()).thenReturn(all);

        // when
        ResultObj<List<UomBankDto>> result = adapterService2.getUom(null);

        // then
        assertThat(result.getData()).containsExactlyElementsOf(all);
        verify(adapterCacheOps).getAllUoms();
        verifyNoInteractions(cacheGetOrLoadService);
    }

    @Test
    @SuppressWarnings("unchecked")
    void getUom_whenCodesNotEmpty_thenUsesCacheGetOrLoadService() {
        // given
        List<String> codes = List.of("EA", "KG");
        List<UomBankDto> loaded = List.of(
                UomBankDto.builder().uomCode("EA").build(),
                UomBankDto.builder().uomCode("KG").build()
        );

        when(cacheGetOrLoadService.fetchData(
                eq(AdapterService2.UOM_BY_CODE),
                eq(codes)
        )).thenReturn((List) loaded);

        // when
        ResultObj<List<UomBankDto>> result = adapterService2.getUom(codes);

        // then
        assertThat(result.getData()).containsExactlyElementsOf(loaded);
        verify(cacheGetOrLoadService).fetchData(AdapterService2.UOM_BY_CODE, codes);
        verifyNoInteractions(adapterCacheOps);
    }

    // ---------- Material types ----------

    @Test
    void getMaterialType_whenIdsNull_thenUsesAllFromCacheOps() {
        // given
        List<MaterialTypeDto> all = List.of(
                MaterialTypeDto.builder().typeId("TYPE1").build()
        );
        when(adapterCacheOps.getAllMaterialTypes()).thenReturn(all);

        // when
        ResultObj<List<MaterialTypeDto>> result = adapterService2.getMaterialType(null);

        // then
        assertThat(result.getData()).containsExactlyElementsOf(all);
        verify(adapterCacheOps).getAllMaterialTypes();
        verifyNoInteractions(cacheGetOrLoadService);
    }

    @Test
    @SuppressWarnings("unchecked")
    void getMaterialType_whenIdsNotEmpty_thenUsesCacheGetOrLoadService() {
        // given
        List<String> ids = List.of("MT1", "MT2");
        List<MaterialTypeDto> loaded = List.of(
                MaterialTypeDto.builder().typeId("MT1").build(),
                MaterialTypeDto.builder().typeId("MT2").build()
        );

        when(cacheGetOrLoadService.fetchData(
                eq(AdapterService2.MATERIAL_TYPE_BY_ID),
                eq(ids)
        )).thenReturn((List) loaded);

        // when
        ResultObj<List<MaterialTypeDto>> result = adapterService2.getMaterialType(ids);

        // then
        assertThat(result.getData()).containsExactlyElementsOf(loaded);
        verify(cacheGetOrLoadService).fetchData(AdapterService2.MATERIAL_TYPE_BY_ID, ids);
        verifyNoInteractions(adapterCacheOps);
    }

    // ---------- Materials ----------

    @Test
    void getMaterial_whenCodesNull_thenUsesAllFromCacheOps() {
        // given
        List<MaterialDto> all = List.of(
            MaterialDto.builder().materialCode("M1").build()
        );
        when(adapterCacheOps.getAllMaterials()).thenReturn(all);

        // when
        ResultObj<List<MaterialDto>> result = adapterService2.getMaterial(null);

        // then
        assertThat(result.getData()).containsExactlyElementsOf(all);
        verify(adapterCacheOps).getAllMaterials();
        verifyNoInteractions(cacheGetOrLoadService);
    }

    @Test
    @SuppressWarnings("unchecked")
    void getMaterial_whenCodesNotEmpty_thenUsesCacheGetOrLoadService() {
        // given
        List<String> codes = List.of("M1", "M2");
        List<MaterialDto> loaded = List.of(
                MaterialDto.builder().materialCode("M1").build(),
                MaterialDto.builder().materialCode("M2").build()
        );

        when(cacheGetOrLoadService.fetchData(
                eq(AdapterService2.MATERIAL_BY_CODE),
                eq(codes)
        )).thenReturn((List) loaded);

        // when
        ResultObj<List<MaterialDto>> result = adapterService2.getMaterial(codes);

        // then
        assertThat(result.getData()).containsExactlyElementsOf(loaded);
        verify(cacheGetOrLoadService).fetchData(AdapterService2.MATERIAL_BY_CODE, codes);
        verifyNoInteractions(adapterCacheOps);
    }
}
```
