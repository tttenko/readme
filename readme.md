```java

@ExtendWith(MockitoExtension.class)
class CacheManagerServiceTest {

    @Mock
    private CacheManager cacheManager;

    @InjectMocks
    private CacheManagerService service;

    @Test
    void getCacheStatus_whenNoCachesAndNoInvalidate_returnsDefaults() {
        // given
        when(cacheManager.getCacheNames()).thenReturn(Collections.emptyList());

        // when
        CacheStatusResponse status = service.getCacheStatus();

        // then
        assertEquals("Not yet completed", status.getLastManualInvalidation());
        assertEquals(CacheConst.TTL, status.getTtl());
        assertNotNull(status.getCaches());
        assertTrue(status.getCaches().isEmpty());
    }

    @Test
    void getCacheStatus_withCaffeineCache_fillsBasicFields() {
        // given
        var nativeCache = Caffeine.newBuilder()
                .recordStats()
                .build();
        nativeCache.put("k1", "v1");

        Cache springCache = new CaffeineCache("testCache", nativeCache);

        when(cacheManager.getCacheNames()).thenReturn(List.of("testCache"));
        when(cacheManager.getCache("testCache")).thenReturn(springCache);

        // when
        CacheStatusResponse status = service.getCacheStatus();

        // then
        assertEquals(1, status.getCaches().size());
        CacheStatusDto cacheStatus = status.getCaches().get(0);

        assertEquals("testCache", cacheStatus.getName());
        assertEquals("CaffeineCache", cacheStatus.getType());
        // ключевой показатель, что статистика реально берётся из nativeCache
        assertEquals(1L, cacheStatus.getEstimatedSize());
    }

    @Test
    void getCacheStatus_withNonCaffeineCache_usesElseBranch() {
        // given
        Cache otherCache = mock(Cache.class);

        when(cacheManager.getCacheNames()).thenReturn(List.of("other"));
        when(cacheManager.getCache("other")).thenReturn(otherCache);

        // when
        CacheStatusResponse status = service.getCacheStatus();

        // then
        assertEquals(1, status.getCaches().size());
        CacheStatusDto cacheStatus = status.getCaches().get(0);

        assertEquals("other", cacheStatus.getName());
        assertEquals(otherCache.getClass().getSimpleName(), cacheStatus.getType());
        // статистика не должна заполняться для non-Caffeine
        assertNull(cacheStatus.getEstimatedSize());
    }

    @Test
    void invalidateCache_clearsAllCachesAndUpdatesLastInvalidate() {
        // given
        Cache cache1 = mock(Cache.class);
        Cache cache2 = mock(Cache.class);

        when(cacheManager.getCacheNames()).thenReturn(List.of("c1", "c2"));
        when(cacheManager.getCache("c1")).thenReturn(cache1);
        when(cacheManager.getCache("c2")).thenReturn(cache2);

        // when
        service.invalidateCache();

        // then: оба кэша очищены
        verify(cache1).clear();
        verify(cache2).clear();

        // после этого статус должен показывать, что инвалидация уже была
        when(cacheManager.getCacheNames()).thenReturn(Collections.emptyList());
        CacheStatusResponse status = service.getCacheStatus();
        assertNotEquals("Not yet completed", status.getLastManualInvalidation());
    }
}
```
