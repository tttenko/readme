```java

// imports для этих тестов:
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.util.List;
import java.util.function.Function;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import org.mockito.Mockito;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;

/**
 * <b>Given</b> CacheManager не знает кеш с заданным именем (возвращает {@code null}).<br>
 * <b>When</b> запрашиваем один ключ через {@code fetchBatch}.<br>
 * <b>Then</b> ветка {@code cache == null}: loader вызывается один раз, элемент возвращается,
 * запись в кеш не производится.
 */
@Test
void givenUnknownCacheName_whenFetchSingle_thenLoaderCalled_andResultReturned() {
  CacheManager manager = Mockito.mock(CacheManager.class);
  when(manager.getCache("missing")).thenReturn(null);

  BatchCacheSupport localBatch = new BatchCacheSupport(manager);

  @SuppressWarnings("unchecked")
  Function<List<String>, List<Tb>> loader =
      (Function<List<String>, List<Tb>>) Mockito.mock(Function.class);
  when(loader.apply(Mockito.anyList())).thenReturn(List.of(new Tb("A", "Bank A")));

  List<Tb> result = localBatch.fetchBatch(
      "missing", List.of("A"), loader, Tb::getCode, Tb.class
  );

  ArgumentCaptor<List<String>> captor =
      (ArgumentCaptor<List<String>>) (ArgumentCaptor<?>) ArgumentCaptor.forClass(List.class);
  verify(loader, times(1)).apply(captor.capture());
  assertEquals(List.of("A"), captor.getValue());

  assertEquals(1, result.size());
  assertEquals("A", result.get(0).getCode());
}

/**
 * <b>Given</b> кеш существует, но {@code cache.get(key, type)} кидает {@link ClassCastException}.<br>
 * <b>When</b> запрашиваем один ключ.<br>
 * <b>Then</b> ветка type-mismatch: выполняется {@code cache.evict(key)}, вызывается loader,
 * загруженный элемент кладётся в кеш и возвращается.
 */
@Test
void givenCacheGetThrowsClassCast_whenFetchSingle_thenEvicted_loaded_andCached() {
  Cache cache = Mockito.mock(Cache.class);
  when(cache.get("A", Tb.class)).thenThrow(new ClassCastException("wrong type"));

  CacheManager manager = Mockito.mock(CacheManager.class);
  when(manager.getCache("tb_by_code")).thenReturn(cache);

  BatchCacheSupport localBatch = new BatchCacheSupport(manager);

  @SuppressWarnings("unchecked")
  Function<List<String>, List<Tb>> loader =
      (Function<List<String>, List<Tb>>) Mockito.mock(Function.class);
  when(loader.apply(Mockito.anyList())).thenReturn(List.of(new Tb("A", "Bank A")));

  List<Tb> result = localBatch.fetchBatch(
      "tb_by_code", List.of("A"), loader, Tb::getCode, Tb.class
  );

  // cache.evict(key) должен быть вызван при ClassCastException
  verify(cache, times(1)).evict("A");
  // загруженное значение положили в кеш
  verify(cache, times(1)).put(eq("A"), any(Tb.class));

  assertEquals(1, result.size());
  assertEquals("A", result.get(0).getCode());
}

/**
 * <b>Given</b> кеш существует, но {@code cache.get(key, type)} кидает произвольный {@link RuntimeException}.<br>
 * <b>When</b> запрашиваем один ключ.<br>
 * <b>Then</b> vetka runtime-failure: loader вызывается, а {@code cache.evict(key)} <i>не</i> вызывается,
 * загруженный элемент кладётся в кеш и возвращается.
 */
@Test
void givenCacheGetThrowsRuntime_whenFetchSingle_thenNoEvict_loaded_andCached() {
  Cache cache = Mockito.mock(Cache.class);
  when(cache.get("A", Tb.class)).thenThrow(new IllegalStateException("boom"));

  CacheManager manager = Mockito.mock(CacheManager.class);
  when(manager.getCache("tb_by_code")).thenReturn(cache);

  BatchCacheSupport localBatch = new BatchCacheSupport(manager);

  @SuppressWarnings("unchecked")
  Function<List<String>, List<Tb>> loader =
      (Function<List<String>, List<Tb>>) Mockito.mock(Function.class);
  when(loader.apply(Mockito.anyList())).thenReturn(List.of(new Tb("A", "Bank A")));

  List<Tb> result = localBatch.fetchBatch(
      "tb_by_code", List.of("A"), loader, Tb::getCode, Tb.class
  );

  verify(cache, never()).evict("A");          // при RuntimeException не эвиктим
  verify(cache, times(1)).put(eq("A"), any()); // загруженное — положили

  assertEquals(1, result.size());
  assertEquals("A", result.get(0).getCode());
}

```
