```java
package your.package.name;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.cache.caffeine.CaffeineCache;
import org.springframework.lang.NonNull;
import org.springframework.lang.Nullable;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

@Slf4j
@Component
@RequiredArgsConstructor
public class BatchCacheSupport {

    private final CacheManager cacheManager;

    /**
     * Возвращает элементы по ключам: хиты берём из кэша, промахи грузим батчем.
     * Для одиночного ключа с Caffeine используется атомарная вычислялка (single-flight).
     */
    @NonNull
    public <T> List<T> fetchBatch(
            @NonNull String cacheName,
            @NonNull List<String> keys,
            @NonNull Function<List<String>, List<T>> loader,
            @NonNull Function<T, String> keyExtractor,
            @NonNull Class<T> type
    ) {
        final List<String> ks = normalize(keys);
        if (ks.isEmpty()) return List.of();

        final Cache cache = cacheManager.getCache(cacheName);

        if (ks.size() == 1) {
            return fetchSingle(cache, ks.get(0), loader, type);
        }

        // multi-key путь: собрать хиты/промахи → загрузить промахи → положить в кэш → собрать ответ
        final Map<String, T> hits = new LinkedHashMap<>();
        final List<String> miss = new ArrayList<>();
        collectHitsAndMisses(cache, ks, type, hits, miss);

        final List<T> loaded = miss.isEmpty() ? List.of() : loader.apply(miss);

        putLoadedToCache(cache, loaded, keyExtractor);

        return orderByOriginal(ks, hits, loaded, keyExtractor);
    }

    /* ========================= helpers ========================= */

    /** Одиночный ключ: Caffeine → атомарная вычислялка, иначе фолбэк. */
    @NonNull
    private <T> List<T> fetchSingle(
            @Nullable Cache cache,
            @NonNull String key,
            @NonNull Function<List<String>, List<T>> loader,
            @NonNull Class<T> type
    ) {
        if (cache instanceof CaffeineCache caffeine) {
            return fetchSingleWithCaffeine(caffeine, key, loader);
        }
        return fetchSingleFallback(cache, key, loader, type);
    }

    /** Single-flight для Caffeine (native computeIfAbsent), исключения не оборачиваются. */
    @NonNull
    @SuppressWarnings("unchecked")
    private <T> List<T> fetchSingleWithCaffeine(
            @NonNull CaffeineCache caffeine,
            @NonNull String key,
            @NonNull Function<List<String>, List<T>> loader
    ) {
        T one = (T) caffeine.getNativeCache().get(key, _unused ->
                safeFirst(loader.apply(List.of(key)))
        );
        return (one == null) ? List.of() : List.of(one);
    }

    /** Фолбэк для других провайдеров: read-through → miss → прямой loader → put. */
    @NonNull
    private <T> List<T> fetchSingleFallback(
            @Nullable Cache cache,
            @NonNull String key,
            @NonNull Function<List<String>, List<T>> loader,
            @NonNull Class<T> type
    ) {
        T one = safeGet(cache, key, type);
        if (one == null) {
            one = safeFirst(loader.apply(List.of(key)));
            if (cache != null && one != null) {
                cache.put(key, one);
                // при желании можно проверить наличие putIfAbsent у конкретного кэша
            }
        }
        return (one == null) ? List.of() : List.of(one);
    }

    /** Сбор хитов и промахов для мульти-кейса. */
    private <T> void collectHitsAndMisses(
            @Nullable Cache cache,
            @NonNull List<String> keys,
            @NonNull Class<T> type,
            @NonNull Map<String, T> hitsOut,
            @NonNull List<String> missOut
    ) {
        for (String key : keys) {
            T cached = safeGet(cache, key, type);
            if (cached != null) {
                hitsOut.put(key, cached);
            } else {
                missOut.add(key);
            }
        }
    }

    /** Кладём загруженные элементы в кэш. */
    private <T> void putLoadedToCache(
            @Nullable Cache cache,
            @NonNull List<T> loaded,
            @NonNull Function<T, String> keyExtractor
    ) {
        if (cache == null || loaded.isEmpty()) return;
        for (T item : loaded) {
            String k = keyExtractor.apply(item);
            if (k != null) cache.put(k, item);
        }
    }

    /** Собираем ответ в порядке исходных ключей из хитов + загруженных. */
    @NonNull
    private <T> List<T> orderByOriginal(
            @NonNull List<String> originalKeys,
            @NonNull Map<String, T> hits,
            @NonNull List<T> loaded,
            @NonNull Function<T, String> keyExtractor
    ) {
        Map<String, T> byKey = new HashMap<>(hits);
        for (T item : loaded) {
            String k = keyExtractor.apply(item);
            if (k != null) byKey.put(k, item);
        }
        return originalKeys.stream()
                .map(byKey::get)
                .filter(Objects::nonNull)
                .toList();
    }

    /** Мягкое чтение из кэша: типобезопасно, с логом, без выброса исключений. */
    @Nullable
    private static <T> T safeGet(@Nullable Cache cache, @NonNull String key, @NonNull Class<T> type) {
        if (cache == null) return null;
        try {
            return cache.get(key, type);
        } catch (ClassCastException ex) {
            log.warn("Cache type mismatch: cache={}, key={}, expected={}",
                    cache.getName(), key, type.getSimpleName());
            cache.evict(key);
            return null;
        } catch (RuntimeException ex) {
            log.debug("Cache get failed: cache={}, key={}, message={}",
                    cache.getName(), key, ex.getMessage());
            return null;
        }
    }

    /** Нормализация ключей: trim, drop null/empty, distinct c сохранением порядка. */
    @NonNull
    private List<String> normalize(@NonNull List<String> keys) {
        return keys.stream()
                .filter(Objects::nonNull)
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .distinct()
                .collect(Collectors.toCollection(ArrayList::new));
    }

    /** Первый элемент списка или null. */
    @Nullable
    private static <T> T safeFirst(@Nullable List<T> list) {
        return (list == null || list.isEmpty()) ? null : list.get(0);
    }
}

```
