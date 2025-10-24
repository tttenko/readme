```java

package com.example.service;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import com.github.benmanes.caffeine.cache.stats.CacheStats;
import java.time.Duration;
import java.util.List;
import java.util.concurrent.*;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.cache.caffeine.CaffeineCache;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit.jupiter.SpringExtension;

/**
 * Тесты проверяют:
 *  - промах → хит,
 *  - sync=true при конкурентных вызовах,
 *  - независимость кешей,
 *  - ручную инвалидацию,
 *  - (если включено recordStats) статистику хитов/промахов.
 *
 * Контекст поднимается заново на каждый тест, чтобы статистика и состояние кешей не "перетекали".
 */
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = { CacheTestConfig.class, BaseCacheDataService.class })
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_EACH_TEST_METHOD)
class BaseCacheDataServiceTest {

  @Autowired
  private BaseCacheDataService service;

  @Autowired
  private CacheManager cacheManager;

  @MockBean
  private TerBankMapper terBankMapper;

  @MockBean
  private TerBankWithRequisiteMapper terBankWithRequisiteMapper;

  @MockBean
  private BaseMasterDataRequestService baseMasterDataRequestService;

  private SearchRequestProperties props;

  @BeforeEach
  void setUp() {
    // Мокаем properties, чтобы сервис мог построить запрос
    props = Mockito.mock(SearchRequestProperties.class);
    when(props.getSlugValueForTerBank()).thenReturn("terbank-slug");
    when(baseMasterDataRequestService.getProperties()).thenReturn(props);

    // Возвращаем "пустой, но валидный" ответ — createResult вернёт пустой список (и это ок для теста кеша)
    when(baseMasterDataRequestService.requestDataWithAttribute(
            eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
        .thenReturn(new GetItemsSearchResponse());

    // Для метода с реквизитами используем тот же заглушечный ответ
    when(baseMasterDataRequestService.requestDataWithAttribute(
            eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
        .thenReturn(new GetItemsSearchResponse());
  }

  // ---------- Вспомогательно ----------

  private CacheStats statsOf(String cacheName) {
    final Cache cache = cacheManager.getCache(cacheName);
    assertNotNull(cache, "Cache " + cacheName + " not found");
    if (cache instanceof CaffeineCache cc) {
      return cc.getNativeCache().stats();
    }
    // Если провайдер будет не Caffeine, просто возвращаем пустую статистику
    return com.github.benmanes.caffeine.cache.stats.StatsCounter.disabledStats().snapshot();
  }

  private void clear(String cacheName) {
    final Cache cache = cacheManager.getCache(cacheName);
    assertNotNull(cache);
    cache.clear();
  }

  // ---------- Тесты ----------

  @Test
  void getAllBanks_shouldCacheByKeyALL() {
    final var first = service.getAllBanks();
    final var second = service.getAllBanks();

    // backend дернули ровно один раз
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));

    // второй вызов — тот же объект (из кеша)
    assertSame(first, second, "Second call must return cached instance");

    // статистика Caffeine: 1 miss, 1 hit
    final var stats = statsOf(TerBankService_2.TB_ALL);
    assertEquals(1, stats.missCount(), "missCount");
    assertEquals(1, stats.hitCount(), "hitCount");
  }

  @Test
  void getAllBanksWithRequisite_shouldCacheByKeyALL() {
    final var first = service.getAllBanksWithRequisite();
    final var second = service.getAllBanksWithRequisite();

    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));

    assertSame(first, second);

    final var stats = statsOf(TerBankService_2.TB_REQ_ALL);
    assertEquals(1, stats.missCount(), "missCount");
    assertEquals(1, stats.hitCount(), "hitCount");
  }

  @Test
  void cachesShouldBeIndependent() {
    // 1) tb_all
    service.getAllBanks();
    service.getAllBanks();

    // 2) tb_req_all
    service.getAllBanksWithRequisite();
    service.getAllBanksWithRequisite();

    // backend — два разных обращения (по одному на каждый кеш)
    verify(baseMasterDataRequestService, times(2))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));

    // независимая статистика по кешам
    assertEquals(1, statsOf(TerBankService_2.TB_ALL).missCount());
    assertEquals(1, statsOf(TerBankService_2.TB_ALL).hitCount());
    assertEquals(1, statsOf(TerBankService_2.TB_REQ_ALL).missCount());
    assertEquals(1, statsOf(TerBankService_2.TB_REQ_ALL).hitCount());
  }

  @Test
  void clearShouldEvict_thenNextCallLoadsAgain() {
    service.getAllBanks();     // miss → load
    clear(TerBankService_2.TB_ALL);

    service.getAllBanks();     // miss → load again

    verify(baseMasterDataRequestService, times(2))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));
  }

  @Test
  void concurrentCallsWithSameKey_shouldTriggerSingleBackendCall_dueToSyncTrue() throws Exception {
    final int threads = 6;
    final ExecutorService pool = Executors.newFixedThreadPool(threads);
    final CountDownLatch ready = new CountDownLatch(threads);
    final CountDownLatch start = new CountDownLatch(1);

    // Эмулируем "долгий" backend и запускаем все потоки одновременно
    when(baseMasterDataRequestService.requestDataWithAttribute(
            eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK)))
        .thenAnswer(inv -> {
          ready.countDown();
          // ждём, пока все дойдут до бэкэнда
          start.await(5, TimeUnit.SECONDS);
          Thread.sleep(150);
          return new GetItemsSearchResponse();
        });

    List<Future<List<TerBankDto>>> futures = new CopyOnWriteArrayList<>();
    for (int i = 0; i < threads; i++) {
      futures.add(pool.submit(() -> service.getAllBanks()));
    }

    // когда все вызовы "подвесились" в лоадере — отпускаем
    assertTrue(ready.await(5, TimeUnit.SECONDS), "threads not ready");
    start.countDown();

    for (Future<?> f : futures) {
      assertNotNull(f.get(3, TimeUnit.SECONDS));
    }

    pool.shutdownNow();

    // backend должен выполниться единожды
    verify(baseMasterDataRequestService, times(1))
        .requestDataWithAttribute(eq("terbank-slug"), isNull(), eq(SearchRequestProperties.Context.BOOK));

    final var stats = statsOf(TerBankService_2.TB_ALL);

    
    assertEquals(1, stats.missCount(), "One compute for all concurrent callers");
    assertEquals(threads - 1, stats.hitCount(), "Rest should hit the cache");
  }
}

/**
 * Сервис работы с ТерБанками с кешированием на Spring Cache (Caffeine).
 *
 * <p>Поддерживаемые ключи кеша:
 * <ul>
 *   <li>{@value #TB_BY_CODE} — банки по кодам tbCode;</li>
 *   <li>{@value #TB_ALL} — все банки;</li>
 *   <li>{@value #TB_REQ_BY_CODE} — банки с реквизитами по кодам;</li>
 *   <li>{@value #TB_REQ_ALL} — все банки с реквизитами.</li>
 * </ul>
 *
 * <p>Публичные методы сохраняют прежние сигнатуры для совместимости с контроллером.</p>
 */

 /**
     * Возвращает список ТерБанков.
     *
     * <p>Если {@code tbCodes} {@code null} или пуст, возвращает все банки (кеш {@value #TB_ALL}).
     * Иначе выполняет пакетную дозагрузку по указанным кодам с использованием кеша
     * по ключу {@value #TB_BY_CODE}.</p>
     *
     * @param tbCodes список кодов TB; может быть {@code null} или пустым
     * @return успешный результат со списком {@link TerBankDto}
     */

     /**
     * Возвращает список ТерБанков с реквизитами.
     *
     * <p>Если {@code tbCodes} {@code null} или пуст, возвращает все банки с реквизитами
     * (кеш {@value #TB_REQ_ALL}). Иначе выполняет пакетную дозагрузку по кодам
     * с использованием кеша по ключу {@value #TB_REQ_BY_CODE}.</p>
     *
     * @param tbCodes список кодов TB; может быть {@code null} или пустым
     * @return успешный результат со списком {@link TerBankWithRequisiteDto}
     */

     /**
     * Загружает данные банков по кодам из мастер-данных (без реквизитов).
     *
     * <p>Формирует запрос к master-data с атрибутом {@code properties.getSlugValueForTerBank()}
     * в контексте {@code SearchRequestProperties.Context.BOOK} и маппит ответ через
     * {@link TerBankMapper}.</p>
     *
     * @param codes список кодов TB; не {@code null}
     * @return список {@link TerBankDto} по найденным кодам
     */

     /**
     * Загружает данные банков с реквизитами по кодам из мастер-данных.
     *
     * <p>Формирует запрос к master-data с атрибутом {@code properties.getSlugValueForTerBank()}
     * в контексте {@code SearchRequestProperties.Context.BOOK} и маппит ответ через
     * {@link TerBankWithRequisiteMapper}.</p>
     *
     * @param codes список кодов TB; не {@code null}
     * @return список {@link TerBankWithRequisiteDto} по найденным кодам
     */

     

```
