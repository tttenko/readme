```java

/**
 * Тесты кэш-операций: проверяем, что {@link SupplierCacheOps#loadBySupplierId(String)}
 * кэшируется по ключу supplierId и не делает повторных запросов к MDM.
 */
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = SupplierCacheOpsTest.Config.class)
@Import(SupplierCacheOps.class)
class SupplierCacheOpsTest {

  @TestConfiguration
  static class Config {
    @Bean CacheManager cacheManager() {
      // простой in-memory менеджер, как в TerBankCacheOpsTest
      var mgr = new ConcurrentMapCacheManager(SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID);
      mgr.setAllowNullValues(true);
      return mgr;
    }
  }

  @Autowired private SupplierCacheOps supplierCacheOps;
  @Autowired private CacheManager cacheManager;

  @MockBean private BaseMasterDataRequestService baseMasterDataRequestService;
  @MockBean private SearchRequestProperties searchRequestProperties;
  @MockBean private SupplierRequisiteMapper supplierRequisiteMapper;

  @BeforeEach
  void setUp() {
    when(searchRequestProperties.getSlugValueForSupplierRequisite()).thenReturn("supplierRequisite");
    when(searchRequestProperties.getAttributeIdForSupplierRequisite()).thenReturn("SupplierId");

    // чистим кэш между тестами
    CacheIntrospection.nativeMap(cacheManager, SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID).clear();
  }

  /** GIVEN кэш пуст, WHEN вызываем дважды с одним ID, THEN MDM дернётся 1 раз, значение закэшировано под ключом ID. */
  @Test
  void givenId_whenCalledTwice_thenBackendOnce_andCachedUnderIdKey() {
    final String supplierId = "S1";
    final GetItemsSearchResponse resp = new GetItemsSearchResponse();

    when(baseMasterDataRequestService.requestDataWithRefItemSlug(
        eq("supplierRequisite"), eq("SupplierId"), eq(List.of(supplierId))
    )).thenReturn(resp);

    final List<BankDto> mapped = List.of(new BankDto());

    try (MockedStatic<BaseMasterDataRequestService> statics =
             mockStatic(BaseMasterDataRequestService.class, CALLS_REAL_METHODS)) {
      statics.when(() ->
          BaseMasterDataRequestService.createResultWithAttribute(eq(resp), eq(supplierRequisiteMapper))
      ).thenReturn(mapped);

      // 1-й вызов — загрузка и кэширование
      var first = supplierCacheOps.loadBySupplierId(supplierId);
      // 2-й вызов — из кэша
      var second = supplierCacheOps.loadBySupplierId(supplierId);

      // backend вызван ровно 1 раз
      verify(baseMasterDataRequestService, times(1))
          .requestDataWithRefItemSlug("supplierRequisite", "SupplierId", List.of("S1"));

      // в кэше есть ключ 'S1'
      Set<?> keys = CacheIntrospection.keys(cacheManager, SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID);
      org.junit.jupiter.api.Assertions.assertTrue(keys.contains("S1"));

      // под ключом лежит именно наш список; между вызовами тот же инстанс (кэш-хит)
      @SuppressWarnings("unchecked")
      List<BankDto> raw = (List<BankDto>) CacheIntrospection.rawValue(
          cacheManager, SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID, "S1");
      org.junit.jupiter.api.Assertions.assertNotNull(raw);
      org.junit.jupiter.api.Assertions.assertSame(mapped, raw);
      org.junit.jupiter.api.Assertions.assertSame(first, second);
    }
  }

  /** Разные ID → раздельные записи в кэше; каждый ID грузится в MDM один раз. */
  @Test
  void givenTwoDifferentIds_whenCall_thenTwoLoads_andTwoCacheEntries() {
    final String s1 = "S1";
    final String s2 = "S2";
    final GetItemsSearchResponse resp1 = new GetItemsSearchResponse();
    final GetItemsSearchResponse resp2 = new GetItemsSearchResponse();

    when(baseMasterDataRequestService.requestDataWithRefItemSlug("supplierRequisite", "SupplierId", List.of(s1)))
        .thenReturn(resp1);
    when(baseMasterDataRequestService.requestDataWithRefItemSlug("supplierRequisite", "SupplierId", List.of(s2)))
        .thenReturn(resp2);

    final List<BankDto> list1 = List.of(new BankDto());
    final List<BankDto> list2 = List.of(new BankDto());

    try (MockedStatic<BaseMasterDataRequestService> statics =
             mockStatic(BaseMasterDataRequestService.class, CALLS_REAL_METHODS)) {
      statics.when(() ->
          BaseMasterDataRequestService.createResultWithAttribute(eq(resp1), eq(supplierRequisiteMapper))
      ).thenReturn(list1);
      statics.when(() ->
          BaseMasterDataRequestService.createResultWithAttribute(eq(resp2), eq(supplierRequisiteMapper))
      ).thenReturn(list2);

      // S1 дважды (второй раз — из кэша), затем S2
      var r11 = supplierCacheOps.loadBySupplierId(s1);
      var r12 = supplierCacheOps.loadBySupplierId(s1);
      var r2  = supplierCacheOps.loadBySupplierId(s2);

      // backend по каждому ID — по одному разу
      verify(baseMasterDataRequestService, times(1))
          .requestDataWithRefItemSlug("supplierRequisite", "SupplierId", List.of("S1"));
      verify(baseMasterDataRequestService, times(1))
          .requestDataWithRefItemSlug("supplierRequisite", "SupplierId", List.of("S2"));

      // в кэше два ключа: S1 и S2
      Set<?> keys = CacheIntrospection.keys(cacheManager, SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID);
      org.junit.jupiter.api.Assertions.assertTrue(keys.containsAll(List.of("S1", "S2")));

      // лежат соответствующие списки
      @SuppressWarnings("unchecked")
      List<BankDto> raw1 = (List<BankDto>) CacheIntrospection.rawValue(
          cacheManager, SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID, "S1");
      @SuppressWarnings("unchecked")
      List<BankDto> raw2 = (List<BankDto>) CacheIntrospection.rawValue(
          cacheManager, SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID, "S2");

      org.junit.jupiter.api.Assertions.assertSame(list1, raw1);
      org.junit.jupiter.api.Assertions.assertSame(list2, raw2);
      org.junit.jupiter.api.Assertions.assertSame(r11, r12);
      org.junit.jupiter.api.Assertions.assertSame(r11, raw1);
      org.junit.jupiter.api.Assertions.assertSame(r2, raw2);
    }
  }
}
```
