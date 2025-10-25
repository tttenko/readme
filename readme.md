```java

@TestConfiguration
@EnableCaching
public class TestCacheConfig {

  @Bean
  @Primary
  public CacheManager cacheManager() {
    // объявляем кеши, которые будем проверять
    ConcurrentMapCacheManager manager = new ConcurrentMapCacheManager(
        TerBankService_2.TB_ALL,
        TerBankService_2.TB_REQ_ALL
    );
    manager.setAllowNullValues(true); // можно включить, не мешает твоему сценарию
    return manager;
  }
}

public final class CacheIntrospection {
  private CacheIntrospection() {}

  public static Set<?> keys(CacheManager cacheManager, String cacheName) {
    return nativeMap(cacheManager, cacheName).keySet();
  }

  public static Object rawValue(CacheManager cacheManager, String cacheName, Object key) {
    return nativeMap(cacheManager, cacheName).get(key);
  }

  private static ConcurrentMap<?, ?> nativeMap(CacheManager cacheManager, String cacheName) {
    Cache cache = cacheManager.getCache(cacheName);
    if (!(cache instanceof ConcurrentMapCache)) {
      throw new IllegalStateException("Expected ConcurrentMapCache for " + cacheName);
    }
    return (ConcurrentMap<?, ?>) ((ConcurrentMapCache) cache).getNativeCache();
  }
}

@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = { TestCacheConfig.class, TerBankCacheOps.class })
class TerBankCacheOpsTest {

  @Autowired private TerBankCacheOps terBankCacheOps;
  @Autowired private CacheManager cacheManager;

  @MockBean private TerBankMapper terBankMapper;
  @MockBean private TerBankWithRequisiteMapper terBankWithRequisiteMapper;
  @MockBean private BaseMasterDataRequestService baseMasterDataRequestService;

  private SearchRequestProperties searchRequestProperties;

  @BeforeEach
  void setUp() {
    searchRequestProperties = Mockito.mock(SearchRequestProperties.class);
    when(searchRequestProperties.getSlugValueForTerBank()).thenReturn("terbank-slug");
    when(baseMasterDataRequestService.getProperties()).thenReturn(searchRequestProperties);

    // детерминированный ответ внешнего сервиса
    GetItemsSearchResponse response = new GetItemsSearchResponse();
    when(baseMasterDataRequestService.requestDataWithAttribute(
        eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
      .thenReturn(response);

    // мапперы возвращают конкретные списки — мы ими и проверим «все данные попали в кеш»
    List<TerBankDto> allBanks = List.of(new TerBankDto(), new TerBankDto());
    when(terBankMapper.map(response)).thenReturn(allBanks);

    List<TerBankWithRequisiteDto> allReq = List.of(new TerBankWithRequisiteDto());
    when(terBankWithRequisiteMapper.map(response)).thenReturn(allReq);
  }

  @Test
  void givenTbAll_whenCalledTwice_thenBackendCalledOnce_andCachedUnderKeyALL() {
    List<TerBankDto> firstCall = terBankCacheOps.getAllBanks();
    List<TerBankDto> secondCall = terBankCacheOps.getAllBanks();

    // внешний сервис дернули один раз
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));

    // нужное имя кеша и ключ
    Set<?> keys = CacheIntrospection.keys(cacheManager, TerBankService_2.TB_ALL);
    assertTrue(keys.contains("ALL"), "cache must contain key 'ALL'");

    // в кеше лежит именно тот список, который вернул маппер (и возвращается тот же инстанс)
    Object raw = CacheIntrospection.rawValue(cacheManager, TerBankService_2.TB_ALL, "ALL");
    assertNotNull(raw);
    assertTrue(raw instanceof List, "cached value must be List");
    assertSame(firstCall, raw);
    assertSame(firstCall, secondCall);
    assertEquals(2, firstCall.size(), "expect all mapped items cached");
  }

  @Test
  void givenTbReqAll_whenCalledTwice_thenBackendCalledOnce_andCachedUnderKeyALL() {
    List<TerBankWithRequisiteDto> firstCall = terBankCacheOps.getAllBanksWithRequisite();
    List<TerBankWithRequisiteDto> secondCall = terBankCacheOps.getAllBanksWithRequisite();

    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));

    Set<?> keys = CacheIntrospection.keys(cacheManager, TerBankService_2.TB_REQ_ALL);
    assertTrue(keys.contains("ALL"));

    Object raw = CacheIntrospection.rawValue(cacheManager, TerBankService_2.TB_REQ_ALL, "ALL");
    assertNotNull(raw);
    assertTrue(raw instanceof List);
    assertSame(firstCall, raw);
    assertSame(firstCall, secondCall);
    assertEquals(1, firstCall.size(), "expect all mapped items cached");
  }

  @Test
  void givenBothCaches_whenLoadedSeparately_thenTheyAreIndependent() {
    terBankCacheOps.getAllBanks();
    terBankCacheOps.getAllBanksWithRequisite();

    // два разных кеша — два отдельных ключевых пространства
    assertTrue(CacheIntrospection.keys(cacheManager, TerBankService_2.TB_ALL).contains("ALL"));
    assertTrue(CacheIntrospection.keys(cacheManager, TerBankService_2.TB_REQ_ALL).contains("ALL"));
  }

  @Test
  void givenConcurrentCalls_whenSyncTrue_thenSingleBackendLoad() throws Exception {
    when(baseMasterDataRequestService.requestDataWithAttribute(
        eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
      .thenAnswer(invocation -> { Thread.sleep(150); return new GetItemsSearchResponse(); });

    ExecutorService pool = Executors.newFixedThreadPool(6);
    try {
      for (Future<List<TerBankDto>> future :
          pool.invokeAll(List.of(
              () -> terBankCacheOps.getAllBanks(),
              () -> terBankCacheOps.getAllBanks(),
              () -> terBankCacheOps.getAllBanks(),
              () -> terBankCacheOps.getAllBanks(),
              () -> terBankCacheOps.getAllBanks(),
              () -> terBankCacheOps.getAllBanks()
          ), 3, TimeUnit.SECONDS)) {
        assertNotNull(future.get(), "each caller must receive a result");
      }
    } finally {
      pool.shutdownNow();
    }

    // из-за sync=true лоадер должен выполниться один раз
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
  }
}


```
