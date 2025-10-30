```java

@ExtendWith(MockitoExtension.class)
class SupplierService2Test {

  // --- зависимости ---
  @Mock private SupplierCacheOps supplierCache;
  @Mock private BatchCacheSupport batchLoad;
  @Mock private SupplierRequisiteMapper supplierRequisiteMapper;
  @Mock private SupplierMapper supplierMapper;
  @Mock private BaseMasterDataRequestService baseMasterDataRequestService;
  @Mock private SearchRequestProperties properties;

  // реальный CacheManager, чтобы крутился кэш как в проде
  @Spy private CaffeineCacheManager cacheManager =
      new CaffeineCacheManager(
          SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID,
          SupplierService2.SUPPLIER_BY_ID,
          SupplierService2.SUPPLIER_BY_INN_KPP
      );

  // сервис — инжектится конструктором автоматически (порядок полей не важен)
  @InjectMocks private SupplierService2 service;

  @BeforeEach
  void setUp() {
    // настроим Caffeine для CacheManager
    cacheManager.setCaffeine(
        Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats()
    );

    // частые стабы свойств, чтобы не дублировать в тестах
    when(properties.getAttributeIdForInn()).thenReturn("innAttr");
    when(properties.getAttributeIdForKpp()).thenReturn("kppAttr");
    when(properties.getSlugValueForCounterparty()).thenReturn("counterpartyDic");
    when(properties.getSlugValueForSupplierRequisite()).thenReturn("supplierRequisite");
    when(properties.getAttributeIdForSupplierRequisite()).thenReturn("SupplierId");
  }
}


  // --- INN+KPP: основной путь через батч ---
  @Test
  void searchCounterpartiesByCriteria_innAndKpp_hitsBatch_noFallback() {
    final String inn = "7700000000";
    final String kpp = "770001001";
    final var dto = mock(CounterpartyDto.class);

    when(batchLoad.fetchBatch(
        eq(SupplierService2.SUPPLIER_BY_INN_KPP),
        anyList(),
        any(),
        any(),
        eq(CounterpartyDto.class))
    ).thenReturn(List.of(dto));

    final var result = service.searchCounterpartiesByCriteria(inn, kpp);

    // батч вызван с корректным ключом
    @SuppressWarnings("unchecked")
    ArgumentCaptor<List<String>> keysCap = ArgumentCaptor.forClass(List.class);
    verify(batchLoad).fetchBatch(
        eq(SupplierService2.SUPPLIER_BY_INN_KPP), keysCap.capture(), any(), any(), eq(CounterpartyDto.class));
    org.junit.jupiter.api.Assertions.assertEquals(
        List.of("inn:7700000000:kpp:770001001"), keysCap.getValue());

    // фолбэк и ручной put в кэш не трогались
    verify(baseMasterDataRequestService, never()).requestDataWithAttribute(anyString(), anyMap());
    // проверим, что кэш пуст по ключу
    Object cached = CacheIntrospection.rawValue(
        cacheManager, SupplierService2.SUPPLIER_BY_INN_KPP, "inn:7700000000:kpp:770001001");
    org.junit.jupiter.api.Assertions.assertNull(cached);

    org.junit.jupiter.api.Assertions.assertEquals(1, result.getData().size());
    org.junit.jupiter.api.Assertions.assertSame(dto, result.getData().get(0));
  }

  // --- Только INN: батч не используется, прямой вызов к МД ---
  @Test
  void searchCounterpartiesByCriteria_innOnly_bypassBatch_callsDirect() {
    final var mdmResp = mock(GetItemsSearchResponse.class);
    when(baseMasterDataRequestService.requestDataWithAttribute(eq("counterpartyDic"), anyMap()))
        .thenReturn(mdmResp);

    // Если createResultWithAttribute статический — подменяем на предсказуемый список
    final List<CounterpartyDto> mapped = List.of(mock(CounterpartyDto.class));
    try (MockedStatic<BaseMasterDataRequestService> statics =
             mockStatic(BaseMasterDataRequestService.class, CALLS_REAL_METHODS)) {
      statics.when(() ->
          BaseMasterDataRequestService.createResultWithAttribute(eq(mdmResp), eq(supplierMapper))
      ).thenReturn(mapped);

      final var result = service.searchCounterpartiesByCriteria("7700000000", null);

      verify(batchLoad, never()).fetchBatch(any(), anyList(), any(), any(), any());
      verify(baseMasterDataRequestService, times(1))
          .requestDataWithAttribute(eq("counterpartyDic"), argThat(m ->
              Objects.equals(m.get("innAttr"), List.of("7700000000")) && !m.containsKey("kppAttr")));

      org.junit.jupiter.api.Assertions.assertEquals(mapped, result.getData());
    }
  }

  // --- INN+KPP: батч промахнулся → фолбэк в МД + ручной put в кэш ---
  @Test
  void searchCounterpartiesByCriteria_innAndKpp_batchMiss_fallbackAndCachePut() {
    final String inn = "7700000000";
    final String kpp = "770001001";
    final String key = "inn:%s:kpp:%s".formatted(inn, kpp);

    when(batchLoad.fetchBatch(
        eq(SupplierService2.SUPPLIER_BY_INN_KPP),
        anyList(),
        any(),
        any(),
        eq(CounterpartyDto.class))
    ).thenReturn(List.of()); // промах

    final var mdmResp = mock(GetItemsSearchResponse.class);
    when(baseMasterDataRequestService.requestDataWithAttribute(eq("counterpartyDic"), anyMap()))
        .thenReturn(mdmResp);

    final List<CounterpartyDto> loaded = List.of(mock(CounterpartyDto.class));
    try (MockedStatic<BaseMasterDataRequestService> statics =
             mockStatic(BaseMasterDataRequestService.class, CALLS_REAL_METHODS)) {
      statics.when(() ->
          BaseMasterDataRequestService.createResultWithAttribute(eq(mdmResp), eq(supplierMapper))
      ).thenReturn(loaded);

      final var result = service.searchCounterpartiesByCriteria(inn, kpp);

      // фолбэк запросил МД с корректной картой критериев
      verify(baseMasterDataRequestService).requestDataWithAttribute(eq("counterpartyDic"),
          argThat(m ->
              Objects.equals(m.get("innAttr"), List.of(inn)) &&
              Objects.equals(m.get("kppAttr"), List.of(kpp))));

      // в кэше появился записанный результат по ключу
      @SuppressWarnings("unchecked")
      List<CounterpartyDto> cached = (List<CounterpartyDto>) CacheIntrospection
          .rawValue(cacheManager, SupplierService2.SUPPLIER_BY_INN_KPP, key);
      org.junit.jupiter.api.Assertions.assertEquals(loaded, cached);
      org.junit.jupiter.api.Assertions.assertEquals(loaded, result.getData());
    }
  }

  // --- Поиск по списку ID: батч вызывается с корректными аргументами ---
  @Test
  void getCounterpartiesById_usesBatchLoad() {
    final List<String> ids = List.of("A", "B");
    final List<CounterpartyDto> dtos = List.of(mock(CounterpartyDto.class), mock(CounterpartyDto.class));

    when(batchLoad.fetchBatch(
        eq(SupplierService2.SUPPLIER_BY_ID),
        eq(ids),
        any(),
        any(),
        eq(CounterpartyDto.class))
    ).thenReturn(dtos);

    final var result = service.getCounterpartiesById(ids);

    verify(batchLoad).fetchBatch(
        eq(SupplierService2.SUPPLIER_BY_ID),
        eq(ids),
        any(),
        any(),
        eq(CounterpartyDto.class)
    );
    verify(baseMasterDataRequestService, never()).requestDataWithAttribute(anyString(), anyList(), any());
    verify(baseMasterDataRequestService, never()).requestDataWithAttribute(anyString(), anyMap());
    org.junit.jupiter.api.Assertions.assertEquals(dtos, result.getData());
  }

  // --- Реквизиты по supplierId: вызывает SupplierCacheOps для валидных id, пропуская пустые ---
  @Test
  void searchSupplierRequisite_filtersBlanks_usesSupplierCacheOps() {
    final BankDto b1 = mock(BankDto.class);
    final BankDto b2 = mock(BankDto.class);
    when(supplierCache.loadBySupplierId("S1")).thenReturn(List.of(b1, b2));

    final var result = service.searchSupplierRequisite(Arrays.asList("  ", null, "S1", ""));

    verify(batchLoad, never()).fetchBatch(any(), anyList(), any(), any(), any());
    verify(supplierCache, times(1)).loadBySupplierId("S1");
    org.junit.jupiter.api.Assertions.assertEquals(List.of(b1, b2), result.getData());
  }
}

```
