```java

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.cache.CacheManager;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Set;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Function;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.anyList;
import static org.mockito.Mockito.*;

class BatchCacheSupportTest {

    /** DTO для тестов. */
    static final class Tb {
        private final String code;
        private final String name;

        Tb(String code, String name) {
            this.code = code;
            this.name = name;
        }
        String getCode() { return code; }
        String getName() { return name; }
    }

    private static final String CACHE_NAME = "tb_by_code";

    private CacheManager cacheManager;
    private BatchCacheSupport batch;

    @BeforeEach
    void setUp() {
        ConcurrentMapCacheManager manager = new ConcurrentMapCacheManager(CACHE_NAME);
        manager.setAllowNullValues(true);
        this.cacheManager = manager;
        this.batch = new BatchCacheSupport(cacheManager);

        // на всякий случай чистим
        CacheIntrospection.nativeMap(cacheManager, CACHE_NAME).clear();
    }

    @Test
    void givenSomeKeysCached_whenFetchBatch_thenOnlyMissesLoaded_andNoDuplicatesInCache() {
        // A уже в кеше
        CacheIntrospection.nativeMap(cacheManager, CACHE_NAME).put("A", new Tb("A", "Bank A"));

        List<String> requested = Arrays.asList("A", "B", "C");

        @SuppressWarnings("unchecked")
        Function<List<String>, List<Tb>> loader = mock(Function.class);
        when(loader.apply(anyList()))
            .thenReturn(List.of(new Tb("B", "Bank B"), new Tb("C", "Bank C")));

        List<Tb> result = batch.fetchBatch(
            CACHE_NAME, requested, loader, Tb::getCode, Tb.class
        );

        // лоадер вызван один раз, только с отсутствующими ["B","C"]
        @SuppressWarnings("unchecked")
        var captor = org.mockito.ArgumentCaptor.forClass(List.class);
        verify(loader, times(1)).apply(captor.capture());
        List<String> actuallyLoaded = captor.getValue();
        assertEquals(Set.of("B", "C"), new LinkedHashSet<>(actuallyLoaded));

        // в кеше три уникальных ключа, без дублей
        Set<?> keys = CacheIntrospection.keys(cacheManager, CACHE_NAME);
        assertEquals(Set.of("A", "B", "C"), keys);

        // результат содержит все три записи
        assertEquals(3, result.size());
        assertNotNull(CacheIntrospection.rawValue(cacheManager, CACHE_NAME, "A"));
        assertNotNull(CacheIntrospection.rawValue(cacheManager, CACHE_NAME, "B"));
        assertNotNull(CacheIntrospection.rawValue(cacheManager, CACHE_NAME, "C"));
    }

    @Test
    void givenAllKeysCached_whenFetchBatch_thenLoaderNotCalled_andPureHit() {
        CacheIntrospection.nativeMap(cacheManager, CACHE_NAME).put("A", new Tb("A", "Bank A"));
        CacheIntrospection.nativeMap(cacheManager, CACHE_NAME).put("B", new Tb("B", "Bank B"));

        List<String> requested = Arrays.asList("A", "B");

        @SuppressWarnings("unchecked")
        Function<List<String>, List<Tb>> loader = mock(Function.class);

        List<Tb> result = batch.fetchBatch(
            CACHE_NAME, requested, loader, Tb::getCode, Tb.class
        );

        verify(loader, never()).apply(anyList());
        assertEquals(2, result.size());
        assertEquals(2, CacheIntrospection.nativeMap(cacheManager, CACHE_NAME).size());
    }

    @Test
    void givenDuplicateKeys_whenFetchBatch_thenLoaderGetsDistinctMisses_andCacheStoresSingleEntryPerKey() {
        List<String> requested = Arrays.asList("A", "A", "B", "B");

        @SuppressWarnings("unchecked")
        Function<List<String>, List<Tb>> loader = mock(Function.class);
        when(loader.apply(anyList()))
            .thenReturn(List.of(new Tb("A", "Bank A"), new Tb("B", "Bank B")));

        List<Tb> result = batch.fetchBatch(
            CACHE_NAME, requested, loader, Tb::getCode, Tb.class
        );

        @SuppressWarnings("unchecked")
        var captor = org.mockito.ArgumentCaptor.forClass(List.class);
        verify(loader, times(1)).apply(captor.capture());
        assertEquals(Set.of("A", "B"), new LinkedHashSet<>(captor.getValue()));

        Set<?> keys = CacheIntrospection.keys(cacheManager, CACHE_NAME);
        assertEquals(Set.of("A", "B"), keys);

        // в результате присутствуют A и B (дедупликация результата — реализация-зависима, мы проверяем наличие)
        Set<String> codes = result.stream().map(Tb::getCode).collect(java.util.stream.Collectors.toSet());
        assertTrue(codes.containsAll(Set.of("A", "B")));
    }

    @Test
    void givenLoaderReturnsExtraOrMissing_whenFetchBatch_thenCacheOnlyRequestedAndFoundItems() {
        // Запросим A,B — лоадер вернёт A и посторонний X; B не вернёт
        List<String> requested = Arrays.asList("A", "B");

        @SuppressWarnings("unchecked")
        Function<List<String>, List<Tb>> loader = mock(Function.class);
        when(loader.apply(anyList()))
            .thenReturn(List.of(new Tb("A", "Bank A"), new Tb("X", "Bank X")));

        List<Tb> result = batch.fetchBatch(
            CACHE_NAME, requested, loader, Tb::getCode, Tb.class
        );

        Set<?> keys = CacheIntrospection.keys(cacheManager, CACHE_NAME);
        assertTrue(keys.contains("A"));
        assertFalse(keys.contains("B"));
        assertFalse(keys.contains("X"));

        Set<String> codes = result.stream().map(Tb::getCode).collect(java.util.stream.Collectors.toSet());
        assertTrue(codes.contains("A"));
        assertFalse(codes.contains("B"));
        assertFalse(codes.contains("X"));
    }

    @Test
    void givenNullsFromLoader_whenFetchBatch_thenNullIgnored_andNoNullInCache() {
        List<String> requested = Arrays.asList("N1", "N2");

        @SuppressWarnings("unchecked")
        Function<List<String>, List<Tb>> loader = mock(Function.class);
        when(loader.apply(anyList()))
            .thenReturn(Arrays.asList(null, new Tb("N2", "Name 2")));

        List<Tb> result = batch.fetchBatch(
            CACHE_NAME, requested, loader, Tb::getCode, Tb.class
        );

        Set<?> keys = CacheIntrospection.keys(cacheManager, CACHE_NAME);
        assertFalse(keys.contains("N1"));
        assertTrue(keys.contains("N2"));

        assertEquals(1, result.size());
        assertEquals("N2", result.get(0).getCode());
    }

    @Test
    void givenOverlappingConcurrentBatches_whenFetchBatch_thenNoNPlusOne_andFewLoaderCallsOnly() throws Exception {
        CountDownLatch startBarrier = new CountDownLatch(1);
        AtomicInteger loaderCalls = new AtomicInteger(0);

        Function<List<String>, List<Tb>> loader = keys -> {
            loaderCalls.incrementAndGet();
            try {
                startBarrier.await(1, TimeUnit.SECONDS);
                Thread.sleep(150);
            } catch (InterruptedException ignored) {}
            List<Tb> out = new ArrayList<>();
            for (String k : keys) out.add(new Tb(k, "Name " + k));
            return out;
        };

        ExecutorService pool = Executors.newFixedThreadPool(2);
        try {
            Callable<List<Tb>> t1 = () -> batch.fetchBatch(
                CACHE_NAME, Arrays.asList("A", "B", "C"), loader, Tb::getCode, Tb.class);
            Callable<List<Tb>> t2 = () -> batch.fetchBatch(
                CACHE_NAME, Arrays.asList("B", "C", "D"), loader, Tb::getCode, Tb.class);

            Future<List<Tb>> f1 = pool.submit(t1);
            Future<List<Tb>> f2 = pool.submit(t2);

            startBarrier.countDown();
            List<Tb> r1 = f1.get(2, TimeUnit.SECONDS);
            List<Tb> r2 = f2.get(2, TimeUnit.SECONDS);

            assertEquals(3, r1.size());
            assertEquals(3, r2.size());

            // в кеше уникальные ключи A,B,C,D (без дублей)
            Set<?> keys = CacheIntrospection.keys(cacheManager, CACHE_NAME);
            assertEquals(Set.of("A", "B", "C", "D"), keys);

            // допускаем максимум пару батчей (по числу конкурентных запросов), но точно не N+1
            assertTrue(loaderCalls.get() <= 2, "Loader must not be called per-key (no N+1)");
        } finally {
            pool.shutdownNow();
        }
    }
}



@Test
void givenSomeKeysCached_whenFetchBatch_thenOnlyMissesLoaded_andNoDuplicatesInCache() {
  // A уже в кеше
  Cache cache = cacheManager.getCache(CACHE_NAME);
  assertNotNull(cache);
  cache.put("A", new Tb("A", "Bank A"));

  List<String> requested = Arrays.asList("A", "B", "C");

  @SuppressWarnings("unchecked")
  Function<List<String>, List<Tb>> loader =
      (Function<List<String>, List<Tb>>) Mockito.mock(Function.class);

  Mockito.when(loader.apply(Mockito.anyList()))
      .thenReturn(Arrays.asList(new Tb("B", "Bank B"), new Tb("C", "Bank C")));

  // действие
  List<Tb> result = batch.fetchBatch(
      CACHE_NAME,
      requested,
      loader,
      Tb::getCode,
      Tb.class
  );

  // лоадер вызван 1 раз и только по отсутствующим ["B","C"]
  @SuppressWarnings("unchecked")
  ArgumentCaptor<List<String>> captor =
      (ArgumentCaptor<List<String>>) (ArgumentCaptor<?>) ArgumentCaptor.forClass(List.class);

  Mockito.verify(loader, Mockito.times(1)).apply(captor.capture());
  List<String> actuallyLoaded = captor.getValue();
  assertEquals(new LinkedHashSet<>(Arrays.asList("B", "C")),
               new LinkedHashSet<>(actuallyLoaded));

  // в кеше три уникальных ключа без дублей
  Set<?> keys = CacheIntrospection.keys(cacheManager, CACHE_NAME);
  assertEquals(Set.of("A", "B", "C"), keys);

  // значения реально лежат в кеше
  assertNotNull(CacheIntrospection.rawValue(cacheManager, CACHE_NAME, "A"));
  assertNotNull(CacheIntrospection.rawValue(cacheManager, CACHE_NAME, "B"));
  assertNotNull(CacheIntrospection.rawValue(cacheManager, CACHE_NAME, "C"));

  // итоговая выборка из метода — 3 элемента
  assertEquals(3, result.size());
}


```
