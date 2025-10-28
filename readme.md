```java

private <T> void assertConcurrentFanInToSingleBackendCall(
        int threads,
        Callable<T> callUnderTest,
        CountDownLatch backendEntered,
        CountDownLatch releaseBackend,
        Runnable verifyBackendInteractions,
        String assertCacheName
) throws Exception {

    ExecutorService pool = Executors.newFixedThreadPool(threads);
    try {
        // запускаем N конкурентных вызовов
        List<Future<T>> futures = java.util.stream.IntStream.range(0, threads)
                .mapToObj(i -> pool.submit(callUnderTest))
                .toList();

        // гарантируем, что хотя бы один поток дошёл до мокнутого бэкенда
        org.junit.jupiter.api.Assertions.assertTrue(
                backendEntered.await(1, java.util.concurrent.TimeUnit.SECONDS),
                "backend must be entered"
        );

        // отпускаем бэкенд — все вызовы схлопнутся в один
        releaseBackend.countDown();

        // убеждаемся, что все получили результат
        for (Future<T> f : futures) {
            org.junit.jupiter.api.Assertions.assertNotNull(f.get());
        }
    } finally {
        pool.shutdownNow();
    }

    // проверяем взаимодействия и ключ в кеше
    verifyBackendInteractions.run();
    org.junit.jupiter.api.Assertions.assertTrue(
            CacheIntrospection.keys(cacheManager, assertCacheName).contains("ALL")
    );
}


@Test
void givenConcurrentCalls_getAllBanks_thenFanInToSingle_requestData_call() throws Exception {
    CountDownLatch entered  = new CountDownLatch(1);
    CountDownLatch release  = new CountDownLatch(1);

    GetItemsSearchResponse ok =
            MasterDataFixtures.okNoAttrResponse("TB001", "Test Bank 1");

    when(baseMasterDataRequestService.requestData(
            eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
        .thenAnswer(inv -> { entered.countDown(); release.await(2, TimeUnit.SECONDS); return ok; });

    Callable<List<TerBankDto>> call = terBankCacheOps::getAllBanks;

    Runnable verify = () -> {
        verify(baseMasterDataRequestService, times(1))
                .requestData(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
        verify(baseMasterDataRequestService, never()).requestDataWithAttribute(anyString(), any(), any());
    };

    assertConcurrentFanInToSingleBackendCall(
            4,                // threads
            call,
            entered,
            release,
            verify,
            TerBankService2.TB_ALL
    );
}


@Test
void givenConcurrentCalls_getAllBanksWithRequisite_thenFanInToSingle_requestDataWithAttribute_call() throws Exception {
    CountDownLatch entered  = new CountDownLatch(1);
    CountDownLatch release  = new CountDownLatch(1);

    GetItemsSearchResponse ok =
            MasterDataFixtures.okWithAttrResponse("TB001", "Req Bank 1", Map.of("inn", "1234567890"));

    when(baseMasterDataRequestService.requestDataWithAttribute(
            eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
        .thenAnswer(inv -> { entered.countDown(); release.await(2, TimeUnit.SECONDS); return ok; });

    Callable<List<TerBankWithRequisiteDto>> call = terBankCacheOps::getAllBanksWithRequisite;

    Runnable verify = () -> {
        verify(baseMasterDataRequestService, times(1))
                .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
        verify(baseMasterDataRequestService, never()).requestData(anyString(), any(), any());
    };

    assertConcurrentFanInToSingleBackendCall(
            4,
            call,
            entered,
            release,
            verify,
            TerBankService2.TB_REQ_ALL
    );
}

/**
 * Проверяет, что несколько конкурентных вызовов схлопываются
 * в один фактический запрос к backend'у, а результат кэшируется.
 *
 * @param threads                 количество потоков, выполняющих параллельные вызовы
 * @param callUnderTest           операция, вызываемая конкурентно
 * @param backendEntered          latch, сигнализирующий о входе в backend
 * @param releaseBackend          latch, разрешающий выход из backend
 * @param verifyBackendInteractions проверка количества обращений к backend
 * @param assertCacheName         имя кэша, в котором ожидается сохранённый результат
 * @param <T>                     тип возвращаемого значения операции
 * @throws Exception              если выполнение потоков или проверок завершилось с ошибкой
 */
```
