```java

/** Универсальный helper: конкурентно дергает call, проверяет backend и наличие ключа в нужном кеше. */
private <T> void assertConcurrentFanInToSingleBackendCall(
    Runnable stubBackend,                       // как застабить backend (с задержкой)
    Callable<T> callUnderTest,                  // какой сервисный метод вызывать
    Runnable verifyBackendInteractions,         // как верифицировать вызовы backend
    String assertCacheName                      // в каком кеше ждать ключ 'ALL'
) throws Exception {

  // изоляция
  Objects.requireNonNull(cacheManager.getCache(TerBankService_2.TB_ALL)).clear();
  Objects.requireNonNull(cacheManager.getCache(TerBankService_2.TB_REQ_ALL)).clear();
  Mockito.clearInvocations(baseMasterDataRequestService, terBankMapper, terBankWithRequisiteMapper);

  // стабы backend
  stubBackend.run();

  // конкурентные вызовы
  ExecutorService executor = Executors.newFixedThreadPool(4);
  try {
    List<Callable<T>> tasks = IntStream.range(0, 4)
        .mapToObj(i -> callUnderTest)
        .toList();

    for (Future<T> f : executor.invokeAll(tasks, 3, TimeUnit.SECONDS)) {
      assertNotNull(f.get());
    }
  } finally {
    executor.shutdownNow();
  }

  // проверка backend-взаимодействий
  verifyBackendInteractions.run();

  // проверка ключа в нужном кеше
  Set<?> keys = CacheIntrospection.keys(cacheManager, assertCacheName);
  assertTrue(keys.contains("ALL"));
}

____________________________________________

@Test
void givenConcurrentCalls_getAllBanks_thenFanInToSingle_requestData_call() throws Exception {
  GetItemsSearchResponse okNoAttr =
      MasterDataFixtures.okNoAttrResponse("TB001", "Test Bank 1");

  Runnable stub = () -> when(baseMasterDataRequestService.requestData(
          eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
      .thenAnswer(inv -> { Thread.sleep(200); return okNoAttr; });

  Callable<List<TerBankDto>> call = () -> terBankCacheOps.getAllBanks();

  Runnable verifyCalls = () -> {
    verify(baseMasterDataRequestService, times(1))
        .requestData(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
    verify(baseMasterDataRequestService, never())
        .requestDataWithAttribute(anyString(), any(), any());
  };

  assertConcurrentFanInToSingleBackendCall(
      stub, call, verifyCalls, TerBankService_2.TB_ALL);
}

_____________________________________________

@Test
void givenConcurrentCalls_getAllBanksWithRequisite_thenFanInToSingle_requestDataWithAttribute_call() throws Exception {
  GetItemsSearchResponse okWithAttr =
      MasterDataFixtures.okWithAttrResponse("TB001", "Req Bank 1", Map.of("inn", "1234567890"));

  Runnable stub = () -> when(baseMasterDataRequestService.requestDataWithAttribute(
          eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
      .thenAnswer(inv -> { Thread.sleep(200); return okWithAttr; });

  Callable<List<TerBankWithRequisiteDto>> call =
      () -> terBankCacheOps.getAllBanksWithRequisite();

  Runnable verifyCalls = () -> {
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
    verify(baseMasterDataRequestService, never())
        .requestData(anyString(), any(), any());
  };

  assertConcurrentFanInToSingleBackendCall(
      stub, call, verifyCalls, TerBankService_2.TB_REQ_ALL);
}


```
