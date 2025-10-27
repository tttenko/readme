```java

@Test
void givenAllKeysCached_whenFetchBatch_thenLoaderNotCalled_andPureHit() {
  // подготовка кеша: кладем A и B через Spring Cache API
  Cache cache = cacheManager.getCache(CACHE_NAME);
  assertNotNull(cache);
  cache.put("A", new Tb("A", "Bank A"));
  cache.put("B", new Tb("B", "Bank B"));

  List<String> requested = Arrays.asList("A", "B");

  @SuppressWarnings("unchecked")
  Function<List<String>, List<Tb>> loader =
      (Function<List<String>, List<Tb>>) Mockito.mock(Function.class);

  // действие
  List<Tb> result = batch.fetchBatch(
      CACHE_NAME,
      requested,
      loader,
      Tb::getCode,
      Tb.class
  );

  // проверка: чистый HIT — лоадер не вызывался
  Mockito.verify(loader, Mockito.never()).apply(Mockito.anyList());
  assertEquals(2, result.size());

  // проверка кеша через твою интроспекцию
  assertEquals(2, CacheIntrospection.nativeMap(cacheManager, CACHE_NAME).size());
  Set<?> keys = CacheIntrospection.keys(cacheManager, CACHE_NAME);
  assertEquals(Set.of("A", "B"), keys);
  assertNotNull(CacheIntrospection.rawValue(cacheManager, CACHE_NAME, "A"));
  assertNotNull(CacheIntrospection.rawValue(cacheManager, CACHE_NAME, "B"));
}


```
