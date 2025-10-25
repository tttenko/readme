```java

GetItemsSearchResponse okNoAttr = buildOkNoAttrResponse(); // твоя утилита из setUp

  when(baseMasterDataRequestService.requestData(
          eq("terbank-slug"),
          isNull(),
          eq(SearchRequestProperties.Context.BOOK)))
      .thenAnswer(invocation -> {
        Thread.sleep(200);               // имитация долгого запроса
        return okNoAttr;
      });

  ExecutorService executor = Executors.newFixedThreadPool(4);
  try {
    List<Callable<List<TerBankDto>>> callables = List.of(
        () -> terBankCacheOps.getAllBanks(),
        () -> terBankCacheOps.getAllBanks(),
        () -> terBankCacheOps.getAllBanks(),
        () -> terBankCacheOps.getAllBanks()
    );
    for (Future<List<TerBankDto>> future : executor.invokeAll(callables, 3, TimeUnit.SECONDS)) {
      assertNotNull(future.get());
    }
  } finally {
    executor.shutdownNow();
  }

  // assert: при sync=true бэкенд дернулся один раз
  verify(baseMasterDataRequestService, times(1))
      .requestData(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
  verify(baseMasterDataRequestService, never())
      .requestDataWithAttribute(anyString(), any(), any());

```
