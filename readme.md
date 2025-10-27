```java

@Test
void givenConcurrentCalls_getAllBanks_thenSingleBackendCall_withoutSleep() throws Exception {
    // — вход в бэкенд замечен
    CountDownLatch backendEntered = new CountDownLatch(1);
    // — когда отпустить бэкенд
    CountDownLatch releaseBackend = new CountDownLatch(1);

    GetItemsSearchResponse ok =
        MasterDataFixtures.okNoAttrResponse("TB001", "Test Bank 1");

    // Блокируем РОВНО тот поток, который реально пойдёт в бэкенд.
    when(baseMasterDataRequestService.requestData(
            eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
        .thenAnswer(inv -> {
            backendEntered.countDown();                 // сигнал: вошли в бэкенд
            releaseBackend.await(2, TimeUnit.SECONDS);  // ждём разрешения продолжить
            return ok;
        });

    int threads = 4;
    ExecutorService pool = Executors.newFixedThreadPool(threads);
    try {
        // Запускаем параллельные обращения в кэш
        List<Future<List<TerBankDto>>> futures = IntStream.range(0, threads)
            .mapToObj(i -> pool.submit(terBankCacheOps::getAllBanks))
            .toList();

        // Дожидаемся, пока «первый» поток реально вошёл в бэкенд,
        // после чего отпускаем его и завершаем все таски
        assertTrue(backendEntered.await(1, TimeUnit.SECONDS), "backend must be entered");
        releaseBackend.countDown();

        for (Future<List<TerBankDto>> f : futures) {
            assertNotNull(f.get());
        }
    } finally {
        pool.shutdownNow();
    }

    // Проверяем, что к бэкенду сходили один раз и только БЕЗ атрибутов
    verify(baseMasterDataRequestService, times(1))
        .requestData(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
    verify(baseMasterDataRequestService, never()).requestDataWithAttribute(anyString(), any(), any());

    // Проверяем ключ в нужном кеше
    assertTrue(CacheIntrospection.keys(cacheManager, TerBankService2.TB_ALL).contains("ALL"));
}


@Test
void givenConcurrentCalls_getAllBanksWithRequisite_thenSingleBackendCall_withoutSleep() throws Exception {
    CountDownLatch backendEntered = new CountDownLatch(1);
    CountDownLatch releaseBackend = new CountDownLatch(1);

    GetItemsSearchResponse ok =
        MasterDataFixtures.okWithAttrResponse("TB001", "Req Bank 1", Map.of("inn", "1234567890"));

    when(baseMasterDataRequestService.requestDataWithAttribute(
            eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
        .thenAnswer(inv -> {
            backendEntered.countDown();
            releaseBackend.await(2, TimeUnit.SECONDS);
            return ok;
        });

    int threads = 4;
    ExecutorService pool = Executors.newFixedThreadPool(threads);
    try {
        List<Future<List<TerBankWithRequisiteDto>>> futures = IntStream.range(0, threads)
            .mapToObj(i -> pool.submit(terBankCacheOps::getAllBanksWithRequisite))
            .toList();

        assertTrue(backendEntered.await(1, TimeUnit.SECONDS), "backend must be entered");
        releaseBackend.countDown();

        for (Future<List<TerBankWithRequisiteDto>> f : futures) {
            assertNotNull(f.get());
        }
    } finally {
        pool.shutdownNow();
    }

    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
    verify(baseMasterDataRequestService, never()).requestData(anyString(), any(), any());

    assertTrue(CacheIntrospection.keys(cacheManager, TerBankService2.TB_REQ_ALL).contains("ALL"));
}


```
