```java

@Slf4j
@Service
@RequiredArgsConstructor
public class CurrencyService2 {

    public static final String CURRENCY_BY_CODE = "currency_by_code";
    public static final String CURRENCY_ALL = "currency_all";

    private final BatchCacheSupport batchLoad;
    private final CurrencyCacheOps currencyCacheOps;

    @Nonnull
    public ResultObj<List<CurrencyDto>> searchCurrenciesByCode(@Nullable final List<String> currencyCodes) {
        final boolean requestAll = (currencyCodes == null || currencyCodes.isEmpty());

        final List<CurrencyDto> data = requestAll
                ? currencyCacheOps.loadAllCurrencies()
                : batchLoad.fetchBatch(
                        CURRENCY_BY_CODE,
                        currencyCodes,
                        currencyCacheOps::loadByCodes,
                        CurrencyDto::getCurrencyCode,
                        CurrencyDto.class
                );

        return getSuccessResponse(data);
    }
}

@Service
@RequiredArgsConstructor
public class CurrencyCacheOps {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final CurrencyMapper currencyMapper;

    @Cacheable(cacheNames = CurrencyService2.CURRENCY_ALL, key = "'ALL'", sync = true)
    @Nonnull
    public List<CurrencyDto> loadAllCurrencies() {
        final GetItemsSearchResponse resp = baseMasterDataRequestService.requestDataByAttributes(
                properties.getSlugValueForCurrency(),
                properties.getCurrencyAttributeId(),
                null
        );
        return createResultWithAttribute(resp, currencyMapper);
    }

    @Nonnull
    public List<CurrencyDto> loadByCodes(@Nonnull final List<String> codes) {
        final GetItemsSearchResponse resp = baseMasterDataRequestService.requestDataByAttributes(
                properties.getSlugValueForCurrency(),
                properties.getCurrencyAttributeId(),
                codes
        );
        return createResultWithAttribute(resp, currencyMapper);
    }
}

@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = CurrencyCacheOpsTest.Config.class)
@Import(CurrencyCacheOps.class)
class CurrencyCacheOpsTest {

    @TestConfiguration
    @EnableCaching(proxyTargetClass = true)
    static class Config {
        @Bean
        CacheManager cacheManager() {
            var mgr = new CaffeineCacheManager(CurrencyService2.CURRENCY_ALL);
            mgr.setCaffeine(
                    Caffeine.newBuilder()
                            .recordStats()
                            .maximumSize(1_000)
            );
            return mgr;
        }
    }

    @Autowired
    private CurrencyCacheOps currencyCacheOps;

    @Autowired
    private CacheManager cacheManager;

    @MockitoBean
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @MockitoBean
    private SearchRequestProperties properties;

    @MockitoBean
    private CurrencyMapper currencyMapper;

    @BeforeEach
    void setUp() {
        when(properties.getSlugValueForCurrency()).thenReturn("currency");
        when(properties.getCurrencyAttributeId()).thenReturn("currencyCode");

        CacheIntrospection.nativeMap(cacheManager, CurrencyService2.CURRENCY_ALL).clear();
    }

    @Test
    void givenEmptyCache_whenGetAll_thenStoresUnderALLKeyAndReturnsData() {
        var resp = new GetItemsSearchResponse();
        var mapped = List.of(
                CurrencyDto.builder().currencyCode("USD").build(),
                CurrencyDto.builder().currencyCode("EUR").build()
        );

        when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", null))
                .thenReturn(resp);

        try (MockedStatic<BaseMasterDataRequestService> statics = mockStatic(BaseMasterDataRequestService.class)) {
            statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
                    .thenReturn(mapped);

            var first = currencyCacheOps.loadAllCurrencies();
            var second = currencyCacheOps.loadAllCurrencies();

            verify(baseMasterDataRequestService, times(1))
                    .requestDataByAttributes("currency", "currencyCode", null);

            assertThat(first).isEqualTo(second).containsExactlyElementsOf(mapped);

            var keys = CacheIntrospection.keys(cacheManager, CurrencyService2.CURRENCY_ALL);
            assertThat(keys).containsExactly("ALL");

            @SuppressWarnings("unchecked")
            var raw = (List<CurrencyDto>) CacheIntrospection.rawValue(cacheManager, CurrencyService2.CURRENCY_ALL, "ALL");
            assertThat(raw).containsExactlyElementsOf(mapped);
        }
    }

    @Test
    void givenAllCached_whenGetAll_thenReturnsFromCacheHit() {
        var resp = new GetItemsSearchResponse();
        var mapped = List.of(CurrencyDto.builder().currencyCode("JPY").build());

        when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", null))
                .thenReturn(resp);

        try (MockedStatic<BaseMasterDataRequestService> statics = mockStatic(BaseMasterDataRequestService.class)) {
            statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
                    .thenReturn(mapped);

            var cache = (CaffeineCache) cacheManager.getCache(CurrencyService2.CURRENCY_ALL);
            var before = cache.getNativeCache().stats();

            currencyCacheOps.loadAllCurrencies();
            currencyCacheOps.loadAllCurrencies();

            var after = cache.getNativeCache().stats();

            assertThat(after.missCount() - before.missCount()).isEqualTo(1);
            assertThat(after.hitCount() - before.hitCount()).isEqualTo(1);

            verify(baseMasterDataRequestService, times(1))
                    .requestDataByAttributes("currency", "currencyCode", null);
        }
    }

    @Test
    void givenCacheCleared_whenGetAll_thenCacheMissAndReloads() {
        var resp = new GetItemsSearchResponse();
        var mapped = List.of(CurrencyDto.builder().currencyCode("GBP").build());

        when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", null))
                .thenReturn(resp);

        try (MockedStatic<BaseMasterDataRequestService> statics = mockStatic(BaseMasterDataRequestService.class, CALLS_REAL_METHODS)) {
            statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
                    .thenReturn(mapped);

            currencyCacheOps.loadAllCurrencies();

            CacheIntrospection.nativeMap(cacheManager, CurrencyService2.CURRENCY_ALL).remove("ALL");

            currencyCacheOps.loadAllCurrencies();

            verify(baseMasterDataRequestService, times(2))
                    .requestDataByAttributes("currency", "currencyCode", null);
        }
    }

    @Test
    void givenCodes_whenLoadByCodes_thenDelegatesToBackendAndMapsResult() {
        var codes = List.of("USD", "EUR");
        var resp = new GetItemsSearchResponse();
        var mapped = List.of(
                CurrencyDto.builder().currencyCode("USD").build(),
                CurrencyDto.builder().currencyCode("EUR").build()
        );

        when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", codes))
                .thenReturn(resp);

        try (MockedStatic<BaseMasterDataRequestService> statics = mockStatic(BaseMasterDataRequestService.class)) {
            statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
                    .thenReturn(mapped);

            var result = currencyCacheOps.loadByCodes(codes);

            verify(baseMasterDataRequestService, times(1))
                    .requestDataByAttributes("currency", "currencyCode", codes);

            assertThat(result).containsExactlyElementsOf(mapped);

            var keys = CacheIntrospection.keys(cacheManager, CurrencyService2.CURRENCY_ALL);
            assertThat(keys).isEmpty();
        }
    }

    @Test
    void givenTwoSequentialCalls_whenLoadByCodes_thenBackendInvokedTwice_noCaching() {
        var codes = List.of("JPY");
        var resp = new GetItemsSearchResponse();
        var mapped = List.of(CurrencyDto.builder().currencyCode("JPY").build());

        when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", codes))
                .thenReturn(resp);

        try (MockedStatic<BaseMasterDataRequestService> statics = mockStatic(BaseMasterDataRequestService.class)) {
            statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
                    .thenReturn(mapped);

            var r1 = currencyCacheOps.loadByCodes(codes);
            var r2 = currencyCacheOps.loadByCodes(codes);

            assertThat(r1).containsExactlyElementsOf(mapped);
            assertThat(r2).containsExactlyElementsOf(mapped);

            verify(baseMasterDataRequestService, times(2))
                    .requestDataByAttributes("currency", "currencyCode", codes);

            var keys = CacheIntrospection.keys(cacheManager, CurrencyService2.CURRENCY_ALL);
            assertThat(keys).isEmpty();
        }
    }
}

---------------------------------------------------------------------2-------------------------------------------------------------------------

@Slf4j
@Service
@RequiredArgsConstructor
public class AdapterService2 {

    public static final String UOM_BY_CODE = "uom_by_code";
    public static final String MATERIAL_TYPE_BY_ID = "material_type_by_id";
    public static final String MATERIAL_BY_CODE = "material_by_code";

    private final BatchCacheSupport batchLoad;
    private final AdapterCacheOps adapterCacheOps;

    @Nonnull
    public ResultObj<List<UomBankDto>> getUom(@Nullable final List<String> uomCodes) {
        final List<UomBankDto> data = (uomCodes == null || uomCodes.isEmpty())
                ? adapterCacheOps.getAllUoms()
                : batchLoad.fetchBatch(
                        UOM_BY_CODE,
                        uomCodes,
                        adapterCacheOps::loadUomsByCodes,
                        UomBankDto::getUomCode,
                        UomBankDto.class
                );
        return getSuccessResponse(data);
    }

    @Nonnull
    public ResultObj<List<MaterialTypeDto>> getMaterialType(@Nullable final List<String> typeIds) {
        final List<MaterialTypeDto> data = (typeIds == null || typeIds.isEmpty())
                ? adapterCacheOps.getAllMaterialTypes()
                : batchLoad.fetchBatch(
                        MATERIAL_TYPE_BY_ID,
                        typeIds,
                        adapterCacheOps::loadMaterialTypesByIds,
                        MaterialTypeDto::getId,
                        MaterialTypeDto.class
                );
        return getSuccessResponse(data);
    }

    @Nonnull
    public ResultObj<List<MaterialDto>> getMaterial(@Nullable final List<String> materialCodes) {
        final List<MaterialDto> data = (materialCodes == null || materialCodes.isEmpty())
                ? adapterCacheOps.getAllMaterials()
                : batchLoad.fetchBatch(
                        MATERIAL_BY_CODE,
                        materialCodes,
                        adapterCacheOps::loadMaterialsByCodes,
                        MaterialDto::getMaterialCode,
                        MaterialDto.class
                );
        return getSuccessResponse(data);
    }
}

@Component
@RequiredArgsConstructor
public class AdapterCacheOps {

    public static final String UOM_ALL = "uom_all";
    public static final String MATERIAL_TYPE_ALL = "material_type_all";
    public static final String MATERIAL_ALL = "material_all";

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final MeasureUnitMapper measureUnitMapper;
    private final MaterialTypeMapper materialTypeMapper;
    private final MaterialMapper materialMapper;

    @Cacheable(cacheNames = UOM_ALL, key = "'ALL'", sync = true)
    @Nonnull
    public List<UomBankDto> getAllUoms() {
        final ItemsSearchCriteriaRequest request = RequestFactory.getByAttrValuesBuilder()
                .dictionaryName(properties.getSlugValueForMeasureUnit())
                .build();
        final GetItemsSearchResponse resp = baseMasterDataRequestService.requestData(
                request,
                SearchRequestProperties.Context.BOOK);

        return createResultWithAttribute(resp, measureUnitMapper);
    }

    @Cacheable(cacheNames = MATERIAL_TYPE_ALL, key = "'ALL'", sync = true)
    @Nonnull
    public List<MaterialTypeDto> getAllMaterialTypes() {
        final ItemsSearchCriteriaRequest request = RequestFactory.getByAttrValuesBuilder()
                .dictionaryName(properties.getSlugValueForMaterialType())
                .build();
        final GetItemsSearchResponse resp = baseMasterDataRequestService.requestData(
                request,
                SearchRequestProperties.Context.TMC);

        return createResult(resp, materialTypeMapper);
    }

    @Cacheable(cacheNames = MATERIAL_ALL, key = "'ALL'", sync = true)
    @Nonnull
    public List<MaterialDto> getAllMaterials() {
        final ItemsSearchCriteriaRequest request = RequestFactory.getByAttrValuesBuilder()
                .dictionaryName(properties.getSlugValueForMaterial())
                .build();
        final GetItemsSearchResponse resp = baseMasterDataRequestService.requestData(
                request,
                SearchRequestProperties.Context.TMC);

        return createResultWithAttribute(resp, materialMapper);
    }

    @Nonnull
    public List<UomBankDto> loadUomsByCodes(@Nonnull final List<String> codes) {
        final var resp = baseMasterDataRequestService.requestDataWithAttribute(
                properties.getSlugValueForMeasureUnit(),
                codes,
                SearchRequestProperties.Context.BOOK);
        return createResultWithAttribute(resp, measureUnitMapper);
    }

    @Nonnull
    public List<MaterialTypeDto> loadMaterialTypesByIds(@Nonnull final List<String> ids) {
        final var resp = baseMasterDataRequestService.requestData(
                properties.getSlugValueForMaterialType(),
                ids,
                SearchRequestProperties.Context.TMC);
        return createResult(resp, materialTypeMapper);
    }

    @Nonnull
    public List<MaterialDto> loadMaterialsByCodes(@Nonnull final List<String> codes) {
        final var resp = baseMasterDataRequestService.requestDataWithAttribute(
                properties.getSlugValueForMaterial(),
                codes,
                SearchRequestProperties.Context.TMC);
        return createResultWithAttribute(resp, materialMapper);
    }
}

```
