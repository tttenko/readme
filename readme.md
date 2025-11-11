```java

@Test
void givenKeyExtractorThrows_whenFetchBatch_thenItemSkipped_andNoCachePut() {
    // кеш чистый
    CacheIntrospection.nativeMap(cacheManager, CACHE_NAME).clear();

    // лоадер возвращает один валидный элемент
    Function<List<String>, List<Tb>> loader = keys -> List.of(new Tb("A", "Bank A"));

    // keyExtractor кидает исключение → попадём в catch в extractKeySafely()
    Function<Tb, String> explodingExtractor = t -> { throw new RuntimeException("boom"); };

    // действие
    List<Tb> result = batch.fetchBatch(
        CACHE_NAME,
        List.of("A"),
        loader,
        explodingExtractor,     // <-- ловим catch
        Tb.class
    );

    // проверки: результата нет, в кеш ничего не положили
    assertTrue(result.isEmpty(), "Result must be empty when keyExtractor throws");
    assertFalse(CacheIntrospection.keys(cacheManager, CACHE_NAME).contains("A"),
        "Cache must not contain A when keyExtractor failed");
    assertNull(CacheIntrospection.rawValue(cacheManager, CACHE_NAME, "A"));
}

```
