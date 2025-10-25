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

package com.example.service;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.*;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.cache.CacheManager;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit.jupiter.SpringExtension;

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
    when(baseMasterDataRequestService.getProperties()).thenReturn(searchRequestProperties);
    when(searchRequestProperties.getSlugValueForTerBank()).thenReturn("terbank-slug");

    // имитация данных, получаемых от внешнего сервиса
    GetItemsSearchResponse response = new GetItemsSearchResponse();
    when(baseMasterDataRequestService.requestDataWithAttribute(
        eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
      .thenReturn(response);

    // мапперы возвращают осмысленные DTO
    when(terBankMapper.mapValuesToDto(anyMap()))
        .thenAnswer(inv -> {
          Map<String, String> values = inv.getArgument(0);
          TerBankDto dto = new TerBankDto();
          dto.setTbCode(values.getOrDefault("slug", "TEST-CODE"));
          dto.setTbName(values.getOrDefault("name", "Test Bank"));
          return dto;
        });

    when(terBankWithRequisiteMapper.mapValuesToDto(anyMap()))
        .thenAnswer(inv -> {
          TerBankWithRequisiteDto dto = new TerBankWithRequisiteDto();
          dto.setTbCode("REQ-TEST-CODE");
          dto.setTbName("Req Bank");
          return dto;
        });
  }

  // ---- Тест 1 ----
  @Test
  void givenTbAll_whenCalledTwice_thenBackendCalledOnce_andCachedUnderKeyALL() {
    // when
    List<TerBankDto> firstCall = terBankCacheOps.getAllBanks();
    List<TerBankDto> secondCall = terBankCacheOps.getAllBanks();

    // then: внешний сервис вызван 1 раз
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));

    // и данные закешированы под ключом "ALL"
    Set<?> keys = CacheIntrospection.keys(cacheManager, TerBankService_2.TB_ALL);
    assertTrue(keys.contains("ALL"), "cache must contain key 'ALL'");

    // тип и содержимое кеша
    Object raw = CacheIntrospection.rawValue(cacheManager, TerBankService_2.TB_ALL, "ALL");
    assertNotNull(raw);
    assertTrue(raw instanceof List<?>);
    assertSame(firstCall, raw, "cached value must be the same instance as method result");
    assertSame(firstCall, secondCall, "second call must hit cache");
  }

  // ---- Тест 2 ----
  @Test
  void givenTbReqAll_whenCalledTwice_thenBackendCalledOnce_andCachedUnderKeyALL() {
    // when
    List<TerBankWithRequisiteDto> firstCall = terBankCacheOps.getAllBanksWithRequisite();
    List<TerBankWithRequisiteDto> secondCall = terBankCacheOps.getAllBanksWithRequisite();

    // then: бэкенд вызван 1 раз
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));

    Set<?> keys = CacheIntrospection.keys(cacheManager, TerBankService_2.TB_REQ_ALL);
    assertTrue(keys.contains("ALL"));

    Object raw = CacheIntrospection.rawValue(cacheManager, TerBankService_2.TB_REQ_ALL, "ALL");
    assertNotNull(raw);
    assertTrue(raw instanceof List<?>);
    assertSame(firstCall, raw);
    assertSame(firstCall, secondCall);
  }

  // ---- Тест 3 ----
  @Test
  void givenBothCaches_whenLoadedSeparately_thenTheyAreIndependent() {
    terBankCacheOps.getAllBanks();
    terBankCacheOps.getAllBanksWithRequisite();

    Set<?> tbAllKeys = CacheIntrospection.keys(cacheManager, TerBankService_2.TB_ALL);
    Set<?> tbReqAllKeys = CacheIntrospection.keys(cacheManager, TerBankService_2.TB_REQ_ALL);

    assertTrue(tbAllKeys.contains("ALL"));
    assertTrue(tbReqAllKeys.contains("ALL"));
    assertNotSame(
        CacheIntrospection.rawValue(cacheManager, TerBankService_2.TB_ALL, "ALL"),
        CacheIntrospection.rawValue(cacheManager, TerBankService_2.TB_REQ_ALL, "ALL"),
        "different caches must store different objects");
  }

  // ---- Тест 4 ----
  @Test
  void givenConcurrentCalls_whenSyncTrue_thenSingleBackendCall() throws Exception {
    when(baseMasterDataRequestService.requestDataWithAttribute(
        eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
      .thenAnswer(inv -> {
        Thread.sleep(150);
        return new GetItemsSearchResponse();
      });

    ExecutorService pool = Executors.newFixedThreadPool(4);
    try {
      List<Callable<List<TerBankDto>>> tasks = List.of(
          () -> terBankCacheOps.getAllBanks(),
          () -> terBankCacheOps.getAllBanks(),
          () -> terBankCacheOps.getAllBanks(),
          () -> terBankCacheOps.getAllBanks()
      );
      for (Future<List<TerBankDto>> future : pool.invokeAll(tasks, 3, TimeUnit.SECONDS)) {
        assertNotNull(future.get(), "each call must return a result");
      }
    } finally {
      pool.shutdownNow();
    }

    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
  }
}


```
