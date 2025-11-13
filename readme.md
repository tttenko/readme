```java

@ExtendWith(MockitoExtension.class)
class BatchCacheSupportTest {

    private static final String CACHE_NAME = "tb_by_code";

    @Mock
    CacheManager cacheManager;

    @Mock
    Cache cache;

    BatchCacheSupport support;

    @BeforeEach
    void setUp() {
        support = new BatchCacheSupport(cacheManager);
        when(cacheManager.getCache(CACHE_NAME)).thenReturn(cache);
    }

    // ===== collectMisses =====

    @Test
    void collectMisses_emptyKeys_returnsEmptyAndDoesNotHitCacheManager() {
        List<String> result = support.collectMisses(CACHE_NAME, List.of(), Tb.class);

        assertTrue(result.isEmpty());
        verifyNoInteractions(cacheManager);
    }

    @Test
    void collectMisses_skipsHitsAndDeduplicatesMisses() {
        when(cache.get("A", Tb.class)).thenReturn(new Tb("A", "Bank A"));
        when(cache.get("B", Tb.class)).thenReturn(null);
        when(cache.get("C", Tb.class)).thenReturn(null);

        List<String> result = support.collectMisses(
                CACHE_NAME,
                List.of("A", "B", "B", "C"),
                Tb.class
        );

        assertEquals(List.of("B", "C"), result);
    }

    @Test
    void collectMisses_cacheNotFound_throwsException() {
        when(cacheManager.getCache(CACHE_NAME)).thenReturn(null);

        assertThrows(
                MdaCacheNotFoundException.class,
                () -> support.collectMisses(CACHE_NAME, List.of("A"), Tb.class)
        );
    }

    // ===== putToCache =====

    @Test
    void putToCache_emptyList_doesNothing() {
        support.putToCache(CACHE_NAME, List.of(), Tb::getCode);

        verifyNoInteractions(cacheManager);
        verifyNoInteractions(cache);
    }

    @Test
    void putToCache_putsNonNullItemsWithKeys() {
        Tb a = new Tb("A", "Bank A");
        Tb b = new Tb("B", "Bank B");

        support.putToCache(CACHE_NAME, List.of(a, b), Tb::getCode);

        verify(cache).put("A", a);
        verify(cache).put("B", b);
        verifyNoMoreInteractions(cache);
    }

    @Test
    void putToCache_skipsNullItems() {
        Tb b = new Tb("B", "Bank B");

        support.putToCache(CACHE_NAME, Arrays.asList(null, b), Tb::getCode);

        verify(cache).put("B", b);
        verify(cache, never()).put(anyString(), isNull());
    }

    @Test
    void putToCache_blankKey_throwsInvalidKeyException() {
        Tb bad = new Tb("", "Bad");

        assertThrows(
                MdaInvalidCacheKeyException.class,
                () -> support.putToCache(CACHE_NAME, List.of(bad), Tb::getCode)
        );

        // убедимся, что в кэш ничего не положили
        verify(cache, never()).put(anyString(), any());
    }

    // ===== readFromCache =====

    @Test
    void readFromCache_returnsNonNullValuesInOrder() {
        Tb a = new Tb("A", "Bank A");

        when(cache.get("A", Tb.class)).thenReturn(a);
        when(cache.get("B", Tb.class)).thenReturn(null);
        when(cache.get("C", Tb.class)).thenReturn(a);

        List<Tb> result = support.readFromCache(
                CACHE_NAME,
                List.of("A", "B", "C"),
                Tb.class
        );

        assertEquals(List.of(a, a), result); // B выпал как null
    }

    @Test
    void readFromCache_cacheNotFound_throwsException() {
        when(cacheManager.getCache(CACHE_NAME)).thenReturn(null);

        assertThrows(
                MdaCacheNotFoundException.class,
                () -> support.readFromCache(CACHE_NAME, List.of("A"), Tb.class)
        );
    }

    // ===== Вспомогательный DTO только для тестов =====

    private static class Tb {
        private final String code;
        private final String name;

        Tb(String code, String name) {
            this.code = code;
            this.name = name;
        }

        String getCode() {
            return code;
        }

        @Override
        public String toString() {
            return "Tb{" +
                    "code='" + code + '\'' +
                    ", name='" + name + '\'' +
                    '}';
        }
    }
}

```
