```java

@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = AdapterCacheOpsTest.Config.class)
@Import(AdapterCacheOps.class)
class AdapterCacheOpsTest {

  // ---- Тестовый контекст ----
  static class Config {
    @Bean CacheManager cacheManager() {
      var mgr = new org.springframework.cache.caffeine.CaffeineCacheManager(
          AdapterCacheOps.UOM_ALL,
          AdapterCacheOps.MATERIAL_TYPE_ALL,
          AdapterCacheOps.MATERIAL_ALL
      );
      mgr.setCaffeine(Caffeine.newBuilder().recordStats().maximumSize(1_000));
      return mgr;
    }
  }

  @Autowired private AdapterCacheOps adapterCacheOps;
  @Autowired private CacheManager cacheManager;

  @MockBean private BaseMasterDataRequestService baseMasterDataRequestService;
  @MockBean private SearchRequestProperties properties;

  @MockBean private MeasureUnitMapper measureUnitMapper;
  @MockBean private MaterialTypeMapper materialTypeMapper;
  @MockBean private MaterialMapper materialMapper;

  @BeforeEach
  void setUp() {
    // словари из настроек
    when(properties.getSlugValueForMeasureUnit()).thenReturn("measureUnit");
    when(properties.getSlugValueForMaterialType()).thenReturn("materialType");
    when(properties.getSlugValueForMaterial()).thenReturn("material");

    // чистим все 3 кэша перед тестом
    nativeMap(cacheManager, AdapterCacheOps.UOM_ALL).clear();
    nativeMap(cacheManager, AdapterCacheOps.MATERIAL_TYPE_ALL).clear();
    nativeMap(cacheManager, AdapterCacheOps.MATERIAL_ALL).clear();
  }

  // ====================== UOM =======================

  @Test
  @DisplayName("UOM: пустой кэш → первый вызов кладёт под ключ 'ALL' и возвращает данные")
  void uom_givenEmptyCache_whenGetAll_thenStoresALLAndReturnsData() {
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(
        UomBankDto.builder().code("EA").build(),
        UomBankDto.builder().code("KG").build()
    );

    when(baseMasterDataRequestService.requestData(any(), eq(SearchRequestProperties.Context.BOOK)))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, measureUnitMapper))
          .thenReturn(mapped);

      var first = adapterCacheOps.getAllUoms();
      var second = adapterCacheOps.getAllUoms();

      // backend дернули 1 раз
      Mockito.verify(baseMasterDataRequestService, times(1))
          .requestData(any(), eq(SearchRequestProperties.Context.BOOK));

      assertThat(first).isEqualTo(second).containsExactlyElementsOf(mapped);

      // ключ 'ALL' в кэше
      assertThat(keys(cacheManager, AdapterCacheOps.UOM_ALL)).containsExactly("ALL");
      @SuppressWarnings("unchecked")
      var raw = (List<UomBankDto>) CacheIntrospection.rawValue(cacheManager, AdapterCacheOps.UOM_ALL, "ALL");
      assertThat(raw).containsExactlyElementsOf(mapped);
    }
  }

  @Test
  @DisplayName("UOM: при попадании из кэша растут hit, а miss не увеличивается")
  void uom_givenAllCached_whenGetAll_thenHitStats() {
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(UomBankDto.builder().code("PC").build());

    when(baseMasterDataRequestService.requestData(any(), eq(SearchRequestProperties.Context.BOOK)))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, measureUnitMapper))
          .thenReturn(mapped);

      var cache = (CaffeineCache) cacheManager.getCache(AdapterCacheOps.UOM_ALL);
      var before = cache.getNativeCache().stats();

      adapterCacheOps.getAllUoms(); // miss + load
      adapterCacheOps.getAllUoms(); // hit

      var after = cache.getNativeCache().stats();
      assertThat(after.missCount() - before.missCount()).isEqualTo(1);
      assertThat(after.hitCount() - before.hitCount()).isEqualTo(1);
    }
  }

  @Test
  @DisplayName("UOM: очистили ключ 'ALL' → следующий вызов снова ходит в backend")
  void uom_givenCacheCleared_whenGetAll_thenMissAndReload() {
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(UomBankDto.builder().code("L").build());

    when(baseMasterDataRequestService.requestData(any(), eq(SearchRequestProperties.Context.BOOK)))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, measureUnitMapper))
          .thenReturn(mapped);

      adapterCacheOps.getAllUoms(); // 1
      nativeMap(cacheManager, AdapterCacheOps.UOM_ALL).remove("ALL");
      adapterCacheOps.getAllUoms(); // 2

      Mockito.verify(baseMasterDataRequestService, times(2))
          .requestData(any(), eq(SearchRequestProperties.Context.BOOK));
    }
  }

  @Test
  @DisplayName("UOM: loadByCodes — делегирует в backend и не кэширует")
  void uom_givenCodes_whenLoadByCodes_thenDelegatesNoCaching() {
    var codes = List.of("EA", "KG");
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(
        UomBankDto.builder().code("EA").build(),
        UomBankDto.builder().code("KG").build()
    );

    when(baseMasterDataRequestService.requestDataWithAttribute(eq("measureUnit"), eq(codes),
        eq(SearchRequestProperties.Context.BOOK)))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, measureUnitMapper))
          .thenReturn(mapped);

      var result = adapterCacheOps.loadUomsByCodes(codes);
      assertThat(result).containsExactlyElementsOf(mapped);

      Mockito.verify(baseMasterDataRequestService, times(1))
          .requestDataWithAttribute(eq("measureUnit"), eq(codes), eq(SearchRequestProperties.Context.BOOK));

      // кэш ALL не трогаем
      assertThat(keys(cacheManager, AdapterCacheOps.UOM_ALL)).isEmpty();
    }
  }

  // ================== MATERIAL TYPES ==================

  @Test
  @DisplayName("MaterialType: кэширование ALL и один поход в backend")
  void materialType_cacheAll_onceBackend() {
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(MaterialTypeDto.builder().code("TYPE1").build());

    when(baseMasterDataRequestService.requestData(any(), eq(SearchRequestProperties.Context.TMC)))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {
      st.when(() -> BaseMasterDataRequestService.createResult(resp, materialTypeMapper))
          .thenReturn(mapped);

      adapterCacheOps.getAllMaterialTypes();
      adapterCacheOps.getAllMaterialTypes();

      Mockito.verify(baseMasterDataRequestService, times(1))
          .requestData(any(), eq(SearchRequestProperties.Context.TMC));

      assertThat(keys(cacheManager, AdapterCacheOps.MATERIAL_TYPE_ALL)).containsExactly("ALL");
    }
  }

  @Test
  @DisplayName("MaterialType: loadByIds — без кэширования")
  void materialType_loadByIds_noCaching() {
    var ids = List.of("MT1", "MT2");
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(MaterialTypeDto.builder().code("MT1").build(),
                         MaterialTypeDto.builder().code("MT2").build());

    when(baseMasterDataRequestService.requestData(eq("materialType"), eq(ids),
        eq(SearchRequestProperties.Context.TMC)))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {
      st.when(() -> BaseMasterDataRequestService.createResult(resp, materialTypeMapper))
          .thenReturn(mapped);

      var result = adapterCacheOps.loadMaterialTypesByIds(ids);
      assertThat(result).containsExactlyElementsOf(mapped);
      assertThat(keys(cacheManager, AdapterCacheOps.MATERIAL_TYPE_ALL)).isEmpty();
    }
  }

  // ================== MATERIALS ==================

  @Test
  @DisplayName("Material: кэширование ALL и один поход в backend")
  void material_cacheAll_onceBackend() {
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(MaterialDto.builder().code("M1").build());

    when(baseMasterDataRequestService.requestData(any(), eq(SearchRequestProperties.Context.TMC)))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, materialMapper))
          .thenReturn(mapped);

      adapterCacheOps.getAllMaterials();
      adapterCacheOps.getAllMaterials();

      Mockito.verify(baseMasterDataRequestService, times(1))
          .requestData(any(), eq(SearchRequestProperties.Context.TMC));

      assertThat(keys(cacheManager, AdapterCacheOps.MATERIAL_ALL)).containsExactly("ALL");
    }
  }

  @Test
  @DisplayName("Material: loadByCodes — делегирует и не кэширует")
  void material_loadByCodes_noCaching() {
    var codes = List.of("M1", "M2");
    var resp = new GetItemsSearchResponse();
    var mapped = List.of(MaterialDto.builder().code("M1").build(),
                         MaterialDto.builder().code("M2").build());

    when(baseMasterDataRequestService.requestDataWithAttribute(eq("material"), eq(codes),
        eq(SearchRequestProperties.Context.TMC)))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, materialMapper))
          .thenReturn(mapped);

      var result = adapterCacheOps.loadMaterialsByCodes(codes);
      assertThat(result).containsExactlyElementsOf(mapped);
      assertThat(keys(cacheManager, AdapterCacheOps.MATERIAL_ALL)).isEmpty();
    }
  }
}

/**
 * Unit-тесты для {@link AdapterCacheOps}.
 * <p>
 * Проверяется корректная работа кэширования Caffeine для справочников:
 * <ul>
 *   <li>UOM (единицы измерения),</li>
 *   <li>MaterialType (типы материалов),</li>
 *   <li>Material (материалы).</li>
 * </ul>
 * Тесты охватывают сценарии кэширования ключа "ALL", подсчёт hit/miss и отсутствие кэширования
 * при выборочной загрузке по кодам или идентификаторам.
 *
 * @author <PRIVATE_PERSON>
 * @since 2025-11
 * @see AdapterCacheOps
 */
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = AdapterCacheOpsTest.Config.class)
@Import(AdapterCacheOps.class)
class AdapterCacheOpsTest {

    /**
     * Проверяет, что при пустом кэше данные загружаются из backend и сохраняются под ключом "ALL".
     *
     * @see AdapterCacheOps#getAllUoms()
     */
    @Test
    void uom_givenEmptyCache_whenGetAll_thenStoresALLAndReturnsData() { /* ... */ }

    /**
     * Проверяет, что при повторном вызове растёт hit, а miss не увеличивается.
     *
     * @see AdapterCacheOps#getAllUoms()
     */
    @Test
    void uom_givenAllCached_whenGetAll_thenHitStats() { /* ... */ }

    /**
     * Проверяет, что после очистки кэша выполняется повторная загрузка данных.
     *
     * @see AdapterCacheOps#getAllUoms()
     */
    @Test
    void uom_givenCacheCleared_whenGetAll_thenMissAndReload() { /* ... */ }

    /**
     * Проверяет, что выборочная загрузка по кодам не использует кэш.
     *
     * @see AdapterCacheOps#loadUomsByCodes(List)
     */
    @Test
    void uom_givenCodes_whenLoadByCodes_thenDelegatesNoCaching() { /* ... */ }

    /**
     * Проверяет кэширование списка типов материалов.
     *
     * @see AdapterCacheOps#getAllMaterialTypes()
     */
    @Test
    void materialType_cacheAll_onceBackend() { /* ... */ }

    /**
     * Проверяет, что выборочная загрузка типов материалов не кэшируется.
     *
     * @see AdapterCacheOps#loadMaterialTypesByIds(List)
     */
    @Test
    void materialType_loadByIds_noCaching() { /* ... */ }

    /**
     * Проверяет кэширование списка материалов.
     *
     * @see AdapterCacheOps#getAllMaterials()
     */
    @Test
    void material_cacheAll_onceBackend() { /* ... */ }

    /**
     * Проверяет, что загрузка материалов по кодам выполняется без кэширования.
     *
     * @see AdapterCacheOps#loadMaterialsByCodes(List)
     */
    @Test
    void material_loadByCodes_noCaching() { /* ... */ }
}


// ⚠️ НЕ new SearchRequestProperties()
  // Настраиваем существующий бин, который используется сервисами
  searchRequestProperties.setSlugValueForMeasureUnit("uom");
  searchRequestProperties.setUomAttributeId("uomCode");

  searchRequestProperties.setSlugValueForMaterial("material");
  searchRequestProperties.setMaterialAttributeId("materialCode");

  searchRequestProperties.setSlugValueForMaterialType("materialType");
  searchRequestProperties.setMaterialTypeId("typeId");
```
