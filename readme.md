```java

/**
 * Сервис для работы с валютами.
 * <p>
 * Предоставляет операции поиска валют по коду или загрузки всех валют из кэша или внешнего источника.
 * Использует {@link CurrencyCacheOps} для взаимодействия с кэшем и {@link BatchCacheSupport} для пакетной загрузки.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class CurrencyService2 {

    public static final String CURRENCY_BY_CODE = "currency_by_code";
    public static final String CURRENCY_ALL = "currency_all";

    private final BatchCacheSupport batchLoad;
    private final CurrencyCacheOps currencyCacheOps;

    /**
     * Ищет валюты по их кодам или возвращает все валюты, если список кодов пустой.
     *
     * @param currencyCodes список кодов валют для поиска (может быть {@code null} или пустым)
     * @return объект результата, содержащий список найденных валют
     * @see CurrencyCacheOps#loadAllCurrencies()
     * @see CurrencyCacheOps#loadByCodes(List)
     */
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


/**
 * Компонент для работы с кэшированными данными валют.
 * <p>
 * Обеспечивает загрузку всех валют или выборочную загрузку по кодам
 * с использованием {@link BaseMasterDataRequestService} и кэшированием результатов.
 */
@Service
@RequiredArgsConstructor
public class CurrencyCacheOps {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final CurrencyMapper currencyMapper;

    /**
     * Загружает все валюты и кэширует результат под ключом {@code "ALL"}.
     * <p>
     * Используется при первичном обращении для получения полного списка валют.
     *
     * @return список всех валют
     * @see CurrencyService2#CURRENCY_ALL
     */
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

    /**
     * Загружает валюты по указанным кодам без кэширования.
     * <p>
     * Используется при выборочных запросах, когда требуется получить конкретные валюты.
     *
     * @param codes список кодов валют, которые необходимо загрузить
     * @return список найденных валют
     * @see BaseMasterDataRequestService#requestDataByAttributes(String, String, List)
     */
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





/**
 * Тесты для {@link CurrencyCacheOps}.
 * <p>
 * Проверяет корректность работы кэширования валют:
 * сохранение, получение из кэша, очистку и повторные запросы.
 */
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = CurrencyCacheOpsTest.Config.class)
@Import(CurrencyCacheOps.class)
class CurrencyCacheOpsTest {

    /**
     * Конфигурация тестового контекста.
     * Настраивает {@link CacheManager} с использованием Caffeine.
     */
    @TestConfiguration
    @EnableCaching(proxyTargetClass = true)
    static class Config {
        /**
         * Создаёт менеджер кэша с лимитом и статистикой.
         *
         * @return настроенный экземпляр {@link CacheManager}
         */
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

    /**
     * Подготавливает окружение перед каждым тестом:
     * мокаются свойства и очищается кэш.
     */
    @BeforeEach
    void setUp() { ... }

    /**
     * Проверяет, что при пустом кэше данные сохраняются под ключом ALL.
     */
    @Test
    void givenEmptyCache_whenGetAll_thenStoresUnderALLKeyAndReturnsData() { ... }

    /**
     * Проверяет, что при наличии данных в кэше запрос выполняется из него (cache hit).
     */
    @Test
    void givenAllCached_whenGetAll_thenReturnsFromCacheHit() { ... }

    /**
     * Проверяет, что после очистки кэша данные снова загружаются из источника.
     */
    @Test
    void givenCacheCleared_whenGetAll_thenCacheMissAndReloads() { ... }

    /**
     * Проверяет загрузку валют по кодам без кэширования.
     */
    @Test
    void givenCodes_whenLoadByCodes_thenDelegatesToBackendAndMapsResult() { ... }

    /**
     * Проверяет, что повторные вызовы loadByCodes не используют кэш.
     */
    @Test
    void givenTwoSequentialCalls_whenLoadByCodes_thenBackendInvokedTwice_noCaching() { ... }
}

/**
 * Сервис для работы с данными справочников единиц измерения и материалов.
 * <p>
 * Предоставляет методы для получения UOM, типов и кодов материалов
 * с использованием кэша или пакетной загрузки.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AdapterService2 {

    public static final String UOM_BY_CODE = "uom_by_code";
    public static final String MATERIAL_TYPE_BY_ID = "material_type_by_id";
    public static final String MATERIAL_BY_CODE = "material_by_code";

    private final BatchCacheSupport batchLoad;
    private final AdapterCacheOps adapterCacheOps;

    /**
     * Получает список единиц измерения.
     * <p>
     * При пустом списке кодов — загружает все данные, иначе делает выборочную загрузку.
     *
     * @param uomCodes список кодов единиц измерения (может быть {@code null})
     * @return список единиц измерения
     */
    @Nonnull
    public ResultObj<List<UomBankDto>> getUom(@Nullable final List<String> uomCodes) { ... }

    /**
     * Получает список типов материалов.
     *
     * @param typeIds список идентификаторов типов материалов (может быть {@code null})
     * @return список типов материалов
     */
    @Nonnull
    public ResultObj<List<MaterialTypeDto>> getMaterialType(@Nullable final List<String> typeIds) { ... }

    /**
     * Получает список материалов.
     *
     * @param materialCodes список кодов материалов (может быть {@code null})
     * @return список материалов
     */
    @Nonnull
    public ResultObj<List<MaterialDto>> getMaterial(@Nullable final List<String> materialCodes) { ... }
}

/**
 * Класс для кэширования и загрузки данных справочников:
 * единиц измерения (UOM), типов и материалов.
 * <p>
 * Использует {@link BaseMasterDataRequestService} для запросов к данным
 * и сохраняет результаты в кэш для оптимизации повторных обращений.
 */
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

    /**
     * Загружает и кэширует все единицы измерения.
     *
     * @return список всех единиц измерения
     */
    @Cacheable(cacheNames = UOM_ALL, key = "'ALL'", sync = true)
    @Nonnull
    public List<UomBankDto> getAllUoms() { ... }

    /**
     * Загружает и кэширует все типы материалов.
     *
     * @return список всех типов материалов
     */
    @Cacheable(cacheNames = MATERIAL_TYPE_ALL, key = "'ALL'", sync = true)
    @Nonnull
    public List<MaterialTypeDto> getAllMaterialTypes() { ... }

    /**
     * Загружает и кэширует все материалы.
     *
     * @return список всех материалов
     */
    @Cacheable(cacheNames = MATERIAL_ALL, key = "'ALL'", sync = true)
    @Nonnull
    public List<MaterialDto> getAllMaterials() { ... }

    /**
     * Загружает единицы измерения по указанным кодам.
     *
     * @param codes список кодов единиц измерения
     * @return список найденных UOM
     */
    @Nonnull
    public List<UomBankDto> loadUomsByCodes(@Nonnull final List<String> codes) { ... }

    /**
     * Загружает типы материалов по их идентификаторам.
     *
     * @param ids список идентификаторов
     * @return список найденных типов материалов
     */
    @Nonnull
    public List<MaterialTypeDto> loadMaterialTypesByIds(@Nonnull final List<String> ids) { ... }

    /**
     * Загружает материалы по их кодам.
     *
     * @param codes список кодов материалов
     * @return список найденных материалов
     */
    @Nonnull
    public List<MaterialDto> loadMaterialsByCodes(@Nonnull final List<String> codes) { ... }
}

---------------_--------------------------------------__----


/**
 * MVC-тесты контроллера валют.
 */
@SpringBootTest(classes = {
        CurrencyController.class,
        CurrencyService2.class,
        CurrencyCacheOps.class,
        CurrencyMapper.class,
        CacheConfig.class,
        BaseMasterDataRequestService.class
})
class CurrencyControllerMvcTest {

    private AutoCloseable closeable;
    private MockMvc mockMvc;
    private ThreadSafeResourceReader reader;

    @MockBean private ObjectMapper mapper;               // если нужен внутри HttpRequestHelper
    @MockBean private SearchRequestProperties properties;
    @MockBean private HttpRequestHelper httpRequestHelper;

    @Autowired private CurrencyController controller;

    @BeforeEach
    void setUp() {
        closeable = MockitoAnnotations.openMocks(this);

        // ВАЖНО: один инстанс свойств и корректные значения.
        when(properties.getSlugValueForCurrency()).thenReturn("currency");
        when(properties.getCurrencyAttributeId()).thenReturn("currencyCode");

        mockMvc = MockMvcBuilders
                .standaloneSetup(controller)
                .setControllerAdvice(new GlobalExceptionHandler())
                .defaultResponseCharacterEncoding(StandardCharsets.UTF_8)
                .build();

        reader = createReader(this);
    }

    @AfterEach
    void tearDown() throws Exception {
        closeable.close();
    }

    @Test
    @DisplayName("given no codes when GET /currency then returns all")
    void givenNoCodes_whenGetAll_thenOk_andAllReturned() throws Exception {
        // given: MD вернёт все валюты
        GetItemsSearchResponse response = reader.readResource(
                "mdresponce/currency/currency-all.json", GetItemsSearchResponse.class);
        mockPostResponse("/v1/items/byAttrValues", response, GetItemsSearchResponse.class, httpRequestHelper);

        // when/then
        checkResult(performGetOk(mockMvc, "/api/v1/info/currency"), 5);
    }

    @Test
    @DisplayName("given code in query when GET /currency?currencyCode=USD then one returned")
    void givenQueryCode_whenGet_thenOk_andOneReturned() throws Exception {
        // given: MD вернёт одну валюту
        GetItemsSearchResponse response = reader.readResource(
                "mdresponce/currency/currency-USD.json", GetItemsSearchResponse.class);
        mockPostResponse("/v1/items/byAttrValues", response, GetItemsSearchResponse.class, httpRequestHelper);

        // when/then
        checkResult(
                performGetOk(mockMvc, "/api/v1/info/currency", Map.of("currencyCode", "USD")),
                1
        );
    }

    @Test
    @DisplayName("given code in path when GET /currency/{code} then one returned")
    void givenPathCode_whenGet_thenOk_andOneReturned() throws Exception {
        // given
        GetItemsSearchResponse response = reader.readResource(
                "mdresponce/currency/currency-USD.json", GetItemsSearchResponse.class);
        mockPostResponse("/v1/items/byAttrValues", response, GetItemsSearchResponse.class, httpRequestHelper);

        // when/then
        checkResult(performGetOk(mockMvc, "/api/v1/info/currency/USD"), 1);
    }

    @Test
    @DisplayName("given absent code when GET /currency/{code} then 404 from MD is propagated")
    void givenUnknownCode_whenGet_thenNotFoundFromMd() throws Exception {
        // given: MD вернёт «not found»
        GetItemsSearchResponse response = reader.readResource(
                "mdresponce/not-found-response.json", GetItemsSearchResponse.class);
        // стаб для проверки статуса внутри BaseMasterDataRequestService
        mockResponse("/v1/references", response, GetItemsSearchResponse.class, httpRequestHelper);
        // и сам вызов поиска по атрибутам
        mockPostResponse("/v1/items/byAttrValues", response, GetItemsSearchResponse.class, httpRequestHelper);

        // when/then: контроллер пробрасывает ошибку из MD
        assertThatThrownBy(() -> performGetOk(mockMvc, "/api/v1/info/currency/9999"))
        .hasCauseInstanceOf(MdaDataNotFoundException.class);
    }
}
```
