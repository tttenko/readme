```java

@Component
@RequiredArgsConstructor
public final class CacheGateway {

  private final CacheManager cacheManager;

  @Nonnull
  public <T> Optional<T> get(@Nonnull String cacheName, @Nonnull String key, @Nonnull Class<T> type) {
    final @Nullable Cache cache = cacheManager.getCache(cacheName);
    return cache == null ? Optional.empty() : Optional.ofNullable(cache.get(key, type));
  }

  public <T> void put(@Nonnull String cacheName, @Nonnull String key, @Nonnull T value) {
    final @Nullable Cache cache = cacheManager.getCache(cacheName);
    if (cache != null) cache.put(key, value);
  }
}



@Component
public final class KeyValidator {
  public boolean isValid(@Nullable String key) {
    return key != null && !key.isBlank();
  }
}


@Slf4j
@UtilityClass
public class Loaders {
  /**
   * Декоратор, гасащий runtime-ошибки и null → пустой список.
   */
  @Nonnull
  public static <T> Function<List<String>, List<T>> safe(@Nonnull Function<List<String>, List<T>> delegate) {
    return keys -> {
      try {
        final var res = delegate.apply(keys);
        return res == null ? List.of() : res;
      } catch (RuntimeException ex) {
        log.warn("Batch loader failed ({} keys): {}", keys.size(), ex.toString());
        return List.of();
      }
    };
  }
}



@Slf4j
@Component
@RequiredArgsConstructor
public final class BatchCacheOrchestrator {

  private final CacheGateway cache;
  private final KeyValidator keyValidator;

  @Nonnull
  public <T> List<T> fetchBatch(
      @Nonnull String cacheName,
      @Nonnull List<String> keys,
      @Nonnull Function<List<String>, List<T>> loader,
      @Nonnull Function<T, String> keyExtractor,
      @Nonnull Class<T> type) {

    if (keys.isEmpty()) return List.of();

    final Map<String, T> hits = new LinkedHashMap<>();
    final List<String> miss = new ArrayList<>();

    // 1) hits/misses
    for (String k : keys) {
      if (!keyValidator.isValid(k)) {
        log.debug("Skip invalid key: '{}'", k);
        continue;
      }
      cache.get(cacheName, k, type).ifPresentOrElse(v -> hits.put(k, v), () -> miss.add(k));
    }

    // 2) дозагрузка (исключения гасит декоратор)
    final var safeLoader = Loaders.safe(loader);
    final List<T> loaded = miss.isEmpty() ? List.of() : safeLoader.apply(miss);

    // 3) запись в кэш
    for (T item : loaded) {
      if (item == null) continue;
      final String k;
      try { k = keyExtractor.apply(item); }
      catch (RuntimeException ex) { log.warn("Key extraction failed for {}: {}", item, ex.toString()); continue; }
      if (keyValidator.isValid(k)) cache.put(cacheName, k, item);
      else log.debug("Skip caching item with blank key: {}", item);
    }

    // 4) сбор результата в порядке входных ключей
    final Map<String, T> byKey = new HashMap<>(hits);
    for (T item : loaded) {
      if (item == null) continue;
      try {
        final String k = keyExtractor.apply(item);
        if (keyValidator.isValid(k)) byKey.put(k, item);
      } catch (RuntimeException ignore) { /* уже залогировано выше */ }
    }

    final List<T> out = new ArrayList<>(keys.size());
    for (String k : keys) {
      final T v = byKey.get(k);
      if (v != null) out.add(v);
    }
    return out;
  }
}
```
