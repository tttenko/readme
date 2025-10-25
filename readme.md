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

import java.util.*;
import java.util.concurrent.*;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.cache.CacheManager;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit.jupiter.SpringExtension;

/**
 * Интеграционный тест кеша для TerBankCacheOps.
 * Проверяет, что данные из BaseMasterDataRequestService загружаются 1 раз,
 * маппятся в DTO и помещаются в правильный кеш с ключом "ALL".
 */
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

    // --- успешный ответ без атрибутов (для getAllBanks) ---
    GetItemsSearchResponse okNoAttr = new GetItemsSearchResponse();
    // messages: [ { semantic: "S", code: "0", description: "OK" } ]
    ResponseMessage okMsg = new ResponseMessage();
    okMsg.setSemantic("S");
    okMsg.setCode("0");
    okMsg.setDescription("OK");
    okNoAttr.setMessages(List.of(okMsg));
    Map<String, String> item1 = new HashMap<>();
    item1.put("slug", "TB001");
    item1.put("name", "Test Bank 1");
    Map<String, Object> pageNoAttr = new HashMap<>();
    pageNoAttr.put("items", List.of(item1));
    okNoAttr.setData(List.of(pageNoAttr));

    when(baseMasterDataRequestService.requestData(
            eq("terbank-slug"),
            isNull(), // у тебя семантика "null => все"
            eq(SearchRequestProperties.Context.BOOK)))
        .thenReturn(okNoAttr);

    // --- успешный ответ с атрибутами (для getAllBanksWithRequisite) ---
    GetItemsSearchResponse okWithAttr = new GetItemsSearchResponse();
    okWithAttr.setMessages(List.of(okMsg));

    Map<String, Object> itemNode = new HashMap<>();
    Map<String, Object> item = new HashMap<>();
    item.put("slug", "TB001");
    item.put("name", "Req Bank 1");
    itemNode.put("item", item);

    Map<String, Object> attrInn = new HashMap<>();
    Map<String, Object> attrMeta = new HashMap<>();
    attrMeta.put("slug", "inn");           // ATTRIBUTE.slug
    attrInn.put("attribute", attrMeta);
    attrInn.put("value", "1234567890");    // значение атрибута

    itemNode.put("values", List.of(attrInn));
    Map<String, Object> pageWithAttr = new HashMap<>();
    pageWithAttr.put("items", List.of(itemNode));
    okWithAttr.setData(List.of(pageWithAttr));

    when(baseMasterDataRequestService.requestDataWithAttribute(
            eq("terbank-slug"),
            isNull(),
            eq(SearchRequestProperties.Context.BOOK)))
        .thenReturn(okWithAttr);

    // --- моки мапперов ---
    when(terBankMapper.mapValuesToDto(anyMap()))
        .thenAnswer(invocation -> {
            Map<String, String> values = invocation.getArgument(0);
            TerBankDto dto = new TerBankDto();
            dto.setTbCode(values.get("slug"));
            dto.setTbName(values.get("name"));
            return dto;
        });

    when(terBankWithRequisiteMapper.mapValuesToDto(anyMap(), anyMap()))
        .thenAnswer(invocation -> {
            Map<String, Object> values = invocation.getArgument(0);
            Map<String, Map<String, Object>> attrs = invocation.getArgument(1);
            TerBankWithRequisiteDto dto = new TerBankWithRequisiteDto();
            dto.setTbCode(String.valueOf(values.get("slug")));
            dto.setTbName(String.valueOf(values.get("name")));
            dto.setInn(String.valueOf(attrs.get("inn").get("value")));
            return dto;
        });
}


  // ---------- TESTS ----------

  @Test
  void givenTbAll_whenCalledTwice_thenBackendCalledOnce_andCachedUnderKeyALL() {
    // when
    List<TerBankDto> firstCall = terBankCacheOps.getAllBanks();
    List<TerBankDto> secondCall = terBankCacheOps.getAllBanks();

    // then
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));

    // cache name / key verification
    Set<?> keys = CacheIntrospection.keys(cacheManager, TerBankService_2.TB_ALL);
    assertTrue(keys.contains("ALL"), "cache must contain key 'ALL'");

    Object raw = CacheIntrospection.rawValue(cacheManager, TerBankService_2.TB_ALL, "ALL");
    assertNotNull(raw);
    assertTrue(raw instanceof List<?>);
    assertSame(firstCall, raw);
    assertSame(firstCall, secondCall);
  }

  @Test
  void givenTbReqAll_whenCalledTwice_thenBackendCalledOnce_andCachedUnderKeyALL() {
    // when
    List<TerBankWithRequisiteDto> firstCall = terBankCacheOps.getAllBanksWithRequisite();
    List<TerBankWithRequisiteDto> secondCall = terBankCacheOps.getAllBanksWithRequisite();

    // then
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));

    // cache name / key verification
    Set<?> keys = CacheIntrospection.keys(cacheManager, TerBankService_2.TB_REQ_ALL);
    assertTrue(keys.contains("ALL"));

    Object raw = CacheIntrospection.rawValue(cacheManager, TerBankService_2.TB_REQ_ALL, "ALL");
    assertNotNull(raw);
    assertTrue(raw instanceof List<?>);
    assertSame(firstCall, raw);
    assertSame(firstCall, secondCall);
  }

  @Test
  void givenBothCaches_whenLoadedSeparately_thenTheyAreIndependent() {
    terBankCacheOps.getAllBanks();
    terBankCacheOps.getAllBanksWithRequisite();

    Set<?> tbAllKeys = CacheIntrospection.keys(cacheManager, TerBankService_2.TB_ALL);
    Set<?> tbReqAllKeys = CacheIntrospection.keys(cacheManager, TerBankService_2.TB_REQ_ALL);

    assertTrue(tbAllKeys.contains("ALL"));
    assertTrue(tbReqAllKeys.contains("ALL"));

    Object tbAll = CacheIntrospection.rawValue(cacheManager, TerBankService_2.TB_ALL, "ALL");
    Object tbReqAll = CacheIntrospection.rawValue(cacheManager, TerBankService_2.TB_REQ_ALL, "ALL");

    assertNotSame(tbAll, tbReqAll, "different caches must store different objects");
  }

  @Test
  void givenConcurrentCalls_whenSyncTrue_thenSingleBackendCall() throws Exception {
    when(baseMasterDataRequestService.requestDataWithAttribute(
        eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
        .thenAnswer(inv -> {
          Thread.sleep(200);
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
        assertNotNull(future.get());
      }
    } finally {
      pool.shutdownNow();
    }

    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
  }
}


```
