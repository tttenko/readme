```java

@Slf4j
@Component
@RequiredArgsConstructor
public class BatchCacheSupport {

    private final CacheManager cacheManager;

    @NonNull
    public <T> List<T> fetchBatch(
            @NonNull final String cacheName,
            @NonNull final List<String> keys,
            @NonNull final Function<List<String>, List<T>> loader,
            @NonNull final Function<T, String> keyExtractor,
            @NonNull final Class<T> type) {

        if (keys.isEmpty()) return List.of();

        final Cache cache = cacheManager.getCache(cacheName);

        final Map<String, T> hits = new LinkedHashMap<>();
        final List<String> miss = new ArrayList<>();

        collectHitsAndMisses(cache, keys, type, hits, miss);

        final List<T> loaded = miss.isEmpty() ? List.of() : safeLoadBatch(loader, miss);

        putLoadedToCache(cache, loaded, keyExtractor);

        return orderByOriginal(keys, hits, loaded, keyExtractor);
    }

    private static <T> List<T> safeLoadBatch(
            @NonNull final Function<List<String>, List<T>> loader,
            @NonNull final List<String> miss) {
        try {
            final List<T> res = loader.apply(miss);
            return (res == null) ? List.of() : res;
        } catch (RuntimeException ex) {
            return List.of();
        }
    }

    private <T> void collectHitsAndMisses(
            @Nullable final Cache cache,
            @NonNull final List<String> keys,
            @NonNull final Class<T> type,
            @NonNull final Map<String, T> hitsOut,
            @NonNull final List<String> missOut) {
        for (String key : keys) {
            if (key == null || key.isBlank()) {
                log.debug("Skip invalid cache key: '{}'", key);
                continue;
            }

            T cached = null;
            if (cache != null) {
                try {
                    cached = cache.get(key, type);
            }

            if (cached != null) hitsOut.put(key, cached);
            else missOut.add(key);
        }
    }

    private <T> void putLoadedToCache(
            @Nullable final Cache cache,
            @NonNull final List<T> loaded,
            @NonNull final Function<T, String> keyExtractor) {
        if (cache == null || loaded.isEmpty()) return;

        for (T item : loaded) {
            if (item == null) continue;

            final String key;
            try {
                key = keyExtractor.apply(item);
            } catch (RuntimeException ex) {
                log.warn("Failed to extract cache key for item {}: {}", item, ex.toString());
                continue;
            }

            if (key == null || key.isBlank()) {
                log.debug("Skip caching item with blank key: {}", item);
                continue;
            }
            cache.put(key, item);
        }
    }

    @NonNull
    private <T> List<T> orderByOriginal(
            @NonNull final List<String> originalKeys,
            @NonNull final Map<String, T> hits,
            @NonNull final List<T> loaded,
            @NonNull final Function<T, String> keyExtractor) {

        final Map<String, T> byKey = new HashMap<>(hits);
        for (T item : loaded) {
            if (item == null) continue;
            final String key;
            try {
                key = keyExtractor.apply(item);
            } catch (RuntimeException ex) {
                log.warn("Failed to extract key for ordering, item {}: {}", item, ex.toString());
                continue;
            }
            if (key == null || key.isBlank()) continue;
            byKey.put(key, item);
        }

        return originalKeys.stream()
                .map(byKey::get)
                .filter(Objects::nonNull)
                .toList();
    }
}


```
