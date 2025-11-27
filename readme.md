```java

@Service
@RequiredArgsConstructor
public class CacheManagerService {

    private static final DateTimeFormatter DATE_FORMAT =
            DateTimeFormatter.ofPattern("HH:mm:ss dd-MM-yyyy");

    private final CacheManager cacheManager;

    // последнее время ручной инвалидации (через /invalidate)
    private volatile LocalDateTime lastManualInvalidate;

    /**
     * Ручная инвалидация всех кэшей Caffeine.
     * Вызывается из контроллера.
     */
    public void invalidateCache() {
        lastManualInvalidate = LocalDateTime.now();

        for (String cacheName : cacheManager.getCacheNames()) {
            Cache cache = cacheManager.getCache(cacheName);
            if (cache != null) {
                cache.clear();       // Spring Cache: очистка кэша целиком
            }
        }
    }

    /**
     * Возвращает агрегированный статус всех кэшей.
     */
   public Map<String, Serializable> getCacheStatus() {
    Map<String, Serializable> result = new LinkedHashMap<>();

    Map<String, Map<String, Serializable>> caches = new LinkedHashMap<>();
    for (String cacheName : cacheManager.getCacheNames()) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            caches.put(cacheName, buildCacheStatus(cache));
        }
    }

    String lastInvalidateStr = lastManualInvalidate == null
            ? "ещё не выполнялась"
            : lastManualInvalidate.format(DATE_FORMAT);

    result.put("Последняя ручная инвалидация", lastInvalidateStr);
    result.put("TTL (минуты)", CacheConst.TTL);
    result.put("Статус кэшей", (Serializable) caches);

    return result;
}

private Map<String, Serializable> buildCacheStatus(Cache springCache) {
    Map<String, Serializable> status = new LinkedHashMap<>();

    if (springCache instanceof CaffeineCache caffeineCache) {
        var nativeCache = caffeineCache.getNativeCache();
        CacheStats stats = nativeCache.stats();

        status.put("Размер (estimatedSize)", nativeCache.estimatedSize());
        status.put("Попадания (hitCount)", stats.hitCount());
        status.put("Промахи (missCount)", stats.missCount());
        status.put("Hit rate", stats.hitRate());
        status.put("Miss rate", stats.missRate());
        status.put("Успешные загрузки (loadSuccess)", stats.loadSuccessCount());
        status.put("Ошибки загрузки (loadFailure)", stats.loadFailureCount());
        status.put("Удаления (evictionCount)", stats.evictionCount());
    } else {
        status.put("Тип", springCache.getClass().getSimpleName());
    }

    return status;
}
}

```
