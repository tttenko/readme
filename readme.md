```java

public <T> List<String> collectMisses(
    @NonNull String cacheName,
    @NonNull List<String> keys,
    @NonNull Class<T> type
) {
    if (keys.isEmpty()) return List.of();

    Cache cache = cacheManager.getCache(cacheName);
    if (cache == null) {
        throw new IllegalStateException("Unknown cache: " + cacheName);
    }

    return keys.stream()
        .filter(k -> cache.get(k, type) == null)
        .distinct()
        .toList();
}

```
