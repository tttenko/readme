```java

@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = CurrencyCacheOpsTest.Config.class)
@Import(CurrencyCacheOps.class)
class CurrencyCacheOpsTest {

  @TestConfiguration
  @EnableCaching
  static class Config {
    @Bean
    CacheManager cacheManager() {
      var mgr = new ConcurrentMapCacheManager(CurrencyService2.CURRENCY_ALL);
      mgr.setAllowNullValues(true);
      return mgr;
    }
  }

  @Autowired private CurrencyCacheOps currencyCacheOps;
  @Autowired private CacheManager cacheManager;

  @MockBean private BaseMasterDataRequestService baseMasterDataRequestService;
  @MockBean private SearchRequestProperties properties;
  @MockBean private CurrencyMapper currencyMapper;

  @BeforeEach
  void setUp() {
    when(properties.getSlugValueForCurrency()).thenReturn("currency");
    when(properties.getCurrencyAttributeId()).thenReturn("currencyCode");
    CacheIntrospection.nativeMap(cacheManager, CurrencyService2.CURRENCY_ALL).clear();
  }

  @Test
  void getAll_cachesResult_andStoresUnderAllKey() {
    var resp = new GetItemsSearchResponse();
    var mapped = of(
        CurrencyDto.builder().currencyCode("USD").build(),
        CurrencyDto.builder().currencyCode("EUR").build()
    );

    when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", null))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> statics =
             mockStatic(BaseMasterDataRequestService.class)) {
      statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
          .thenReturn(mapped);

      var first  = currencyCacheOps.getAll();
      var second = currencyCacheOps.getAll();

      verify(baseMasterDataRequestService, times(1))
          .requestDataByAttributes("currency", "currencyCode", null);

      // assertThat
      assertThat(first).isEqualTo(second).containsExactlyElementsOf(mapped);

      Set<?> keys = CacheIntrospection.keys(cacheManager, CurrencyService2.CURRENCY_ALL);
      assertThat(keys).contains("ALL");

      @SuppressWarnings("unchecked")
      var raw = (List<CurrencyDto>) CacheIntrospection.rawValue(
          cacheManager, CurrencyService2.CURRENCY_ALL, "ALL");

      assertThat(raw).isNotNull().containsExactlyElementsOf(mapped);
    }
  }

  @Test
  void getAll_syncTrue_avoidsStampede_onParallelCalls() throws Exception {
    var resp = new GetItemsSearchResponse();
    var mapped = of(CurrencyDto.builder().currencyCode("JPY").build());

    when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", null))
        .thenAnswer(inv -> { Thread.sleep(150); return resp; });

    try (MockedStatic<BaseMasterDataRequestService> statics =
             mockStatic(BaseMasterDataRequestService.class)) {
      statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
          .thenReturn(mapped);

      int threads = 8;
      var start = new CountDownLatch(1);
      var done  = new CountDownLatch(threads);
      var pool  = Executors.newFixedThreadPool(threads);

      for (int i = 0; i < threads; i++) {
        pool.submit(() -> {
          try { start.await(); currencyCacheOps.getAll(); }
          catch (InterruptedException ignored) {}
          finally { done.countDown(); }
        });
      }

      start.countDown();
      assertThat(done.await(5, TimeUnit.SECONDS)).isTrue();
      pool.shutdownNow();

      verify(baseMasterDataRequestService, times(1))
          .requestDataByAttributes("currency", "currencyCode", null);

      var keys = CacheIntrospection.keys(cacheManager, CurrencyService2.CURRENCY_ALL);
      assertThat(keys).containsExactly("ALL");
    }
  }

  @Test
  void loadByCodes_callsBackend_andDoesNotTouchAllCache() {
    var codes = of("USD", "EUR");
    var resp  = new GetItemsSearchResponse();
    var mapped = of(
        CurrencyDto.builder().currencyCode("USD").build(),
        CurrencyDto.builder().currencyCode("EUR").build()
    );

    when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", codes))
        .thenReturn(resp);

    try (MockedStatic<BaseMasterDataRequestService> statics =
             mockStatic(BaseMasterDataRequestService.class)) {
      statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
          .thenReturn(mapped);

      var out = currencyCacheOps.loadByCodes(codes);

      verify(baseMasterDataRequestService, times(1))
          .requestDataByAttributes("currency", "currencyCode", codes);

      // assertThat
      assertThat(out).containsExactlyElementsOf(mapped);
      assertThat(CacheIntrospection.keys(cacheManager, CurrencyService2.CURRENCY_ALL)).isEmpty();
    }
  }
}

------------------------------------------------------------------------------------------------------------------------------------------------------

@Test
void getAll_syncTrue_avoidsStampede_onParallelCalls_withoutSleep() throws Exception {
  // given
  var resp   = new GetItemsSearchResponse();
  var mapped = List.of(CurrencyDto.builder().currencyCode("JPY").build());

  // латчи вместо Thread.sleep
  var entered = new CountDownLatch(1);   // сигнал: лоадер вошёл
  var release = new CountDownLatch(1);   // разрешаем завершиться

  when(baseMasterDataRequestService.requestDataByAttributes("currency", "currencyCode", null))
      .thenAnswer(inv -> {
        entered.countDown();                     // сообщаем, что вошли в лоадер
        release.await(3, TimeUnit.SECONDS);      // ждём разрешения (c таймаутом)
        return resp;
      });

  try (MockedStatic<BaseMasterDataRequestService> statics =
           mockStatic(BaseMasterDataRequestService.class)) {
    statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper))
        .thenReturn(mapped);

    int threads = 8;
    var pool = Executors.newFixedThreadPool(threads);
    var futures = new ArrayList<Future<List<CurrencyDto>>>(threads);

    // when: одновременно дергаем кэшируемый метод
    for (int i = 0; i < threads; i++) {
      futures.add(pool.submit(() -> currencyCacheOps.getAll()));
    }

    // убеждаемся, что лоадер реально стартовал, и только потом "будим" его
    assertThat(entered.await(2, TimeUnit.SECONDS)).isTrue();
    release.countDown();

    // then: все потоки получили одинаковый результат
    for (var f : futures) {
      assertThat(f.get(2, TimeUnit.SECONDS)).containsExactlyElementsOf(mapped);
    }

    pool.shutdownNow();

    // один вызов бекенда благодаря sync=true + Caffeine
    verify(baseMasterDataRequestService, times(1))
        .requestDataByAttributes("currency", "currencyCode", null);

    var keys = CacheIntrospection.keys(cacheManager, CurrencyService2.CURRENCY_ALL);
    assertThat(keys.contains("ALL")).isTrue();
  }
}


```
