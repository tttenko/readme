```java
package com.example.cache;

import jakarta.annotation.Nonnull;
import jakarta.annotation.Nullable;
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class BatchCacheSupport {

  private final CacheManager cacheManager;

  /**
   * Возвращает элементы по ключам: берёт хиты из кеша и одним вызовом loader грузит промахи.
   * Загруженные кладёт в кеш, используя keyExtractor.
   */
  @Nonnull
  public <T> List<T> fetchBatch(@Nonnull String cacheName,
                                @Nonnull List<String> keys,
                                @Nonnull Function<List<String>, List<T>> loader,
                                @Nonnull Function<T, String> keyExtractor,
                                @Nonnull Class<T> type) {

    final Cache cache = cacheManager.getCache(cacheName);
    final Map<String, T> hits = new LinkedHashMap<>();
    final List<String> miss = new ArrayList<>();

    for (String key : keys) {
      final T cached = get(cache, key, type);
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

    return keys.stream().map(byKey::get).filter(Objects::nonNull).collect(Collectors.toList());
  }

  @Nullable
  private static <T> T get(@Nullable Cache cache, @Nonnull String key, @Nonnull Class<T> type) {
    if (cache == null) return null;
    try {
      return cache.get(key, type);
    } catch (ClassCastException e) {
      log.warn("Cache type mismatch: cache={}, key={}, expected={}", cache.getName(), key, type.getSimpleName());
      return null;
    }
  }

  /** Нормализация ключей: trim, drop empty, distinct с сохранением порядка. */
  @Nonnull
  public static List<String> normalize(@Nonnull List<String> keys) {
    return keys.stream()
        .filter(Objects::nonNull)
        .map(String::trim)
        .filter(s -> !s.isEmpty())
        .distinct()
        .collect(Collectors.toCollection(ArrayList::new));
  }
}

```
