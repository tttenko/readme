```java

/**
 * Given: кеш пуст, запрошен один ключ {@code A}.
 * When: loader возвращает ровно один элемент.
 * Then: fallback кладёт элемент в кеш и возвращает singleton.
 */
@Test
void givenSingleKeyMiss_whenLoaderReturnsOne_thenItemCachedAndReturned() {
  CacheIntrospection.nativeMap(cacheManager, CACHE_NAME).clear();

  List<String> requested = List.of("A");

  @SuppressWarnings("unchecked")
  Function<List<String>, List<Tb>> loader =
      (Function<List<String>, List<Tb>>) Mockito.mock(Function.class);
  Mockito.when(loader.apply(Mockito.anyList()))
      .thenReturn(List.of(new Tb("A", "Bank A")));

  List<Tb> result = batch.fetchBatch(
      CACHE_NAME, requested, loader, Tb::getCode, Tb.class
  );

  ArgumentCaptor<List<String>> captor =
      (ArgumentCaptor<List<String>>) (ArgumentCaptor<?>) ArgumentCaptor.forClass(List.class);
  Mockito.verify(loader, Mockito.times(1)).apply(captor.capture());
  assertEquals(List.of("A"), captor.getValue());

  assertEquals(1, result.size());
  assertEquals("A", result.get(0).getCode());
  assertTrue(CacheIntrospection.keys(cacheManager, CACHE_NAME).contains("A"));
  assertNotNull(CacheIntrospection.rawValue(cacheManager, CACHE_NAME, "A"));
}


/**
 * Given: в кеше уже лежит {@code A}, запрошен один ключ {@code A}.
 * Then: чистый hit в fallback-ветке — loader не вызывается.
 */
@Test
void givenSingleKeyHit_whenFetchBatch_thenLoaderNotCalled_andReturnsFromCache() {
  Cache cache = cacheManager.getCache(CACHE_NAME);
  assertNotNull(cache);
  cache.put("A", new Tb("A", "Bank A"));

  @SuppressWarnings("unchecked")
  Function<List<String>, List<Tb>> loader =
      (Function<List<String>, List<Tb>>) Mockito.mock(Function.class);

  List<Tb> result = batch.fetchBatch(
      CACHE_NAME, List.of("A"), loader, Tb::getCode, Tb.class
  );

  Mockito.verify(loader, Mockito.never()).apply(Mockito.anyList());
  assertEquals(1, result.size());
  assertEquals("A", result.get(0).getCode());
}


/**
 * Given: кеш пуст, запрошен один ключ {@code B}.
 * When: loader возвращает пустой список.
 * Then: fallback возвращает пустой список и не пишет в кеш.
 */
@Test
void givenSingleKeyMiss_whenLoaderReturnsEmpty_thenEmptyResult_andNoCachePut() {
  List<String> requested = List.of("B");

  @SuppressWarnings("unchecked")
  Function<List<String>, List<Tb>> loader =
      (Function<List<String>, List<Tb>>) Mockito.mock(Function.class);
  Mockito.when(loader.apply(Mockito.anyList())).thenReturn(List.of());

  List<Tb> result = batch.fetchBatch(
      CACHE_NAME, requested, loader, Tb::getCode, Tb.class
  );

  ArgumentCaptor<List<String>> captor =
      (ArgumentCaptor<List<String>>) (ArgumentCaptor<?>) ArgumentCaptor.forClass(List.class);
  Mockito.verify(loader, Mockito.times(1)).apply(captor.capture());
  assertEquals(List.of("B"), captor.getValue());

  assertTrue(result.isEmpty());
  assertFalse(CacheIntrospection.keys(cacheManager, CACHE_NAME).contains("B"));
}


/**
 * Given: кеш пуст, запрошен один ключ {@code N1}.
 * When: loader возвращает список, содержащий {@code null}.
 * Then: fallback игнорирует {@code null}, кеш остаётся пустым, результат — пустой.
 */
@Test
void givenSingleKeyMiss_whenLoaderReturnsNullItem_thenEmptyResult_andNoCachePut() {
  List<String> requested = List.of("N1");

  @SuppressWarnings("unchecked")
  Function<List<String>, List<Tb>> loader =
      (Function<List<String>, List<Tb>>) Mockito.mock(Function.class);
  Mockito.when(loader.apply(Mockito.anyList()))
      .thenReturn(Arrays.asList((Tb) null));

  List<Tb> result = batch.fetchBatch(
      CACHE_NAME, requested, loader, Tb::getCode, Tb.class
  );

  ArgumentCaptor<List<String>> captor =
      (ArgumentCaptor<List<String>>) (ArgumentCaptor<?>) ArgumentCaptor.forClass(List.class);
  Mockito.verify(loader, Mockito.times(1)).apply(captor.capture());
  assertEquals(List.of("N1"), captor.getValue());

  assertTrue(result.isEmpty());
  assertFalse(CacheIntrospection.keys(cacheManager, CACHE_NAME).contains("N1"));
}



```
