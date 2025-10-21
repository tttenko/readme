```java
@Nonnull
public <T> List<T> fetchBatch(@Nonnull String cacheName,
                              @Nonnull List<String> keys,
                              @Nonnull Function<List<String>, List<T>> loader,
                              @Nonnull Function<T, String> keyExtractor,
                              @Nonnull Class<T> type) {

  final List<String> ks = normalize(keys);
  final Cache cache = cacheManager.getCache(cacheName);

  // one-flight для одиночного ключа
  if (ks.size() == 1) {
    final String k = ks.get(0);
    final T one = (cache == null)
        ? safeFirst(loader.apply(List.of(k)))
        : cache.get(k, () -> safeFirst(loader.apply(List.of(k))));
    return (one == null) ? List.of() : List.of(one);
  }

  // дальше — твоя текущая логика хитов/промахов
  final Map<String, T> hits = new LinkedHashMap<>();
  final List<String> miss = new ArrayList<>();
  for (String key : ks) {
    final T cached = (cache != null) ? cache.get(key, type) : null;
    if (cached != null) hits.put(key, cached); else miss.add(key);
  }
  final List<T> loaded = miss.isEmpty() ? List.of() : loader.apply(miss);

  if (cache != null) {
    for (T item : loaded) {
      final String k = keyExtractor.apply(item);
      if (k != null) cache.put(k, item);
    }
  }

  final Map<String, T> byKey = new HashMap<>(hits);
  for (T item : loaded) {
    final String k = keyExtractor.apply(item);
    if (k != null) byKey.put(k, item);
  }
  return ks.stream().map(byKey::get).filter(Objects::nonNull).toList();
}

@Nullable
private static <T> T safeFirst(List<T> list) {
  return (list == null || list.isEmpty()) ? null : list.get(0);
}

```
