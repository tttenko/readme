```java

@Test
void givenTwoDifferentIds_whenCall_thenTwoLoads_andTwoCacheEntries() {
  final String s1 = "S1";
  final String s2 = "S2";

  final GetItemsSearchResponse resp1 = new GetItemsSearchResponse();
  final GetItemsSearchResponse resp2 = new GetItemsSearchResponse();

  // backend по каждому ID возвращает свой респонс
  when(baseMasterDataRequestService.requestDataWithRefItemSlug("supplierRequisite", "SupplierId", List.of(s1)))
      .thenReturn(resp1);
  when(baseMasterDataRequestService.requestDataWithRefItemSlug("supplierRequisite", "SupplierId", List.of(s2)))
      .thenReturn(resp2);

  // маппинг -> предсказуемые списки (важно для сравнения по содержимому)
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

    // S1 два раза (2-й — из кэша), затем S2
    final var r11 = supplierCacheOps.loadBySupplierId(s1);
    final var r12 = supplierCacheOps.loadBySupplierId(s1);
    final var r2  = supplierCacheOps.loadBySupplierId(s2);

    // backend по каждому ID — ровно один раз
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithRefItemSlug("supplierRequisite", "SupplierId", List.of("S1"));
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithRefItemSlug("supplierRequisite", "SupplierId", List.of("S2"));

    // в кэше два ключа
    final Set<?> keys = CacheIntrospection.keys(cacheManager, SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID);
    org.junit.jupiter.api.Assertions.assertTrue(keys.containsAll(List.of("S1", "S2")));

    // извлекаем из кэша и сравниваем по содержимому (а не по ссылке)
    @SuppressWarnings("unchecked")
    final List<BankDto> raw1 = (List<BankDto>) CacheIntrospection.rawValue(
        cacheManager, SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID, "S1");
    @SuppressWarnings("unchecked")
    final List<BankDto> raw2 = (List<BankDto>) CacheIntrospection.rawValue(
        cacheManager, SupplierService2.SUPPLIER_REQ_BY_SUPPLIER_ID, "S2");

    // r11/r12 — одинаковое содержимое; кэш хранит то же содержимое
    org.junit.jupiter.api.Assertions.assertIterableEquals(list1, r11);
    org.junit.jupiter.api.Assertions.assertIterableEquals(list1, r12);
    org.junit.jupiter.api.Assertions.assertIterableEquals(list1, raw1);

    // для второго ключа аналогично
    org.junit.jupiter.api.Assertions.assertIterableEquals(list2, r2);
    org.junit.jupiter.api.Assertions.assertIterableEquals(list2, raw2);
  }

```
