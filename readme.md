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

/**
 * Утилита для «батчевой» работы с кэшем.
 * <p>
 * Основная идея: для набора ключей берём хиты из кэша, а промахи загружаем
 * одним вызовом переданного загрузчика (batch loader). После загрузки
 * результаты кладутся в кэш и возвращаются в порядке исходных ключей.
 * <p>
 * Особый случай для одиночного ключа и {@link CaffeineCache}: используется
 * нативная атомарная вычислялка (single-flight) через
 * {@code nativeCache.get(key, mappingFn)} — это исключает параллельные
 * дубль-вызовы загрузчика по одному и тому же ключу и не заворачивает
 * доменные исключения в {@code Cache.ValueRetrievalException}.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class BatchCacheSupport {

    private final CacheManager cacheManager;

    /**
     * Получить значения по набору ключей, используя кэш и «батчевую» догрузку промахов.
     *
     * <h3>Поведение</h3>
     * <ul>
     *   <li>Ключи предварительно нормализуются: {@code trim}, отбрасываются {@code null}/пустые, оставляется исходный порядок без дубликатов.</li>
     *   <li>Для одиночного ключа и {@link CaffeineCache} выполняется атомарная вычислялка (single-flight) без обёрток Spring Cache.</li>
     *   <li>Для остальных случаев собираются хиты/промахи: промахи загружаются одним вызовом {@code loader.apply(miss)}.</li>
     *   <li>Загруженные элементы кладутся в кэш, ключ для записи берётся из {@code keyExtractor}.</li>
     *   <li>Результат упорядочивается в точном соответствии с исходным списком ключей.</li>
     * </ul>
     *
     * <h3>Исключения</h3>
     * <ul>
     *   <li>Доменные исключения, брошенные {@code loader}, пробрасываются «как есть». Метод намеренно не
     *   заворачивает их в {@code Cache.ValueRetrievalException}.</li>
     *   <li>Ошибки чтения из кэша (не совпал тип, внутренний рантайм) не приводят к падению: запись просто
     *   считается промахом (лог на уровне warn/debug) и значение будет догружено.</li>
     * </ul>
     *
     * <h3>Потоки</h3>
     * <ul>
     *   <li>С {@link CaffeineCache} для одиночного ключа обеспечивается single-flight.</li>
     *   <li>Для мульти-ключевого пути атомарность на уровне кэша зависит от провайдера. При необходимости
     *   используйте нативные API конкретной реализации или внешний коалесинг.</li>
     * </ul>
     *
     * @param cacheName    имя кэша в {@link CacheManager}
     * @param keys         список ключей (может содержать {@code null}/пустые строки — они будут отброшены)
     * @param loader       функция-загрузчик, принимающая список промахов и возвращающая загруженные элементы
     * @param keyExtractor функция, извлекающая ключ кэша из элемента результата
     * @param type         ожидаемый тип значения в кэше (для типобезопасного {@code cache.get})
     * @param <T>          тип элемента результата
     * @return список найденных/загруженных элементов в порядке исходных ключей; невозможные/отсутствующие ключи будут опущены
     * @throws RuntimeException любое доменное исключение, брошенное {@code loader} (не перехватывается намеренно)
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

    /**
     * Получение одного элемента по ключу: если кэш — {@link CaffeineCache}, используется нативный single-flight,
     * иначе — фолбэк через «прочитать → при промахе загрузить → положить».
     *
     * @param cache   кэш (может быть {@code null}, тогда кэширующая часть будет пропущена)
     * @param key     ключ
     * @param loader  загрузчик одного элемента (будет вызван с {@code List.of(key)})
     * @param type    ожидаемый тип значения
     * @param <T>     тип результата
     * @return список из одного элемента или пустой список, если элемента нет
     * @throws RuntimeException доменное исключение из {@code loader}
     */
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

    /**
     * Одиночная загрузка для {@link CaffeineCache} с гарантиями single-flight.
     * <p>
     * Используется {@code nativeCache.get(key, mappingFn)} — вычисление и помещение значения выполняются атомарно,
     * параллельные запросы по тому же ключу ждут готовый результат. Исключения из {@code loader} пробрасываются
     * «как есть».
     *
     * @param caffeine экземпляр {@link CaffeineCache}
     * @param key      ключ
     * @param loader   загрузчик (будет вызван с {@code List.of(key)})
     * @param <T>      тип результата
     * @return список из одного элемента или пустой список, если элемента нет
     * @throws RuntimeException доменное исключение из {@code loader}
     */
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

    /**
     * Одиночная загрузка для прочих провайдеров кэша.
     * <p>
     * Алгоритм: попытаться прочитать → при промахе вызвать {@code loader} → при успехе положить в кэш → вернуть.
     * Ошибки чтения из кэша не приводят к падению и трактуются как промах (с логированием).
     *
     * @param cache  кэш (может быть {@code null})
     * @param key    ключ
     * @param loader загрузчик (будет вызван с {@code List.of(key)})
     * @param type   ожидаемый тип значения
     * @param <T>    тип результата
     * @return список из одного элемента или пустой список, если элемента нет
     * @throws RuntimeException доменное исключение из {@code loader}
     */
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
                // при необходимости можно заменить на putIfAbsent у конкретной реализации
            }
        }
        return (one == null) ? List.of() : List.of(one);
    }

    /**
     * Разделяет исходные ключи на «хиты» (нашлись в кэше) и «промахи» (нужно загрузить).
     * <p>
     * Ошибки чтения из кэша не вызывают падение: ключ рассматривается как промах (с логированием).
     *
     * @param cache   кэш (может быть {@code null})
     * @param keys    нормализованный список ключей
     * @param type    ожидаемый тип значения
     * @param hitsOut мапа для записи найденных значений (ключ → значение)
     * @param missOut список ключей-промахов
     * @param <T>     тип значения
     */
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

    /**
     * Кладёт загруженные элементы в кэш, если он присутствует, используя {@code keyExtractor}.
     * <p>
     * Если {@code keyExtractor} вернул {@code null}, элемент пропускается.
     *
     * @param cache        кэш (может быть {@code null})
     * @param loaded       список загруженных элементов
     * @param keyExtractor функция получения ключа из элемента
     * @param <T>          тип элемента
     */
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

    /**
     * Собирает итоговый список результатов в порядке исходных ключей.
     * <p>
     * Приоритет элементов из «хитов» сохраняется, загруженные дополняют мапу по ключу.
     * Ключи, для которых значение отсутствует, опускаются.
     *
     * @param originalKeys исходный нормализованный порядок ключей
     * @param hits         найденные в кэше значения (ключ → значение)
     * @param loaded       элементы, загруженные из {@code loader}
     * @param keyExtractor функция получения ключа из элемента
     * @param <T>          тип элемента
     * @return список элементов в порядке {@code originalKeys} без {@code null}
     */
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

    /**
     * Безопасное чтение из кэша с типобезопасной десериализацией и «мягкой» обработкой ошибок.
     * <p>
     * Возвращает {@code null} в случаях:
     * <ul>
     *   <li>кэш отсутствует ({@code cache == null});</li>
     *   <li>тип сохранённого значения не совпадает с {@code type} (ключ будет удалён через {@code evict});</li>
     *   <li>кэш бросил рантайм-исключение (событие залогировано, но не прерывает поток выполнения).</li>
     * </ul>
     *
     * @param cache кэш (может быть {@code null})
     * @param key   ключ
     * @param type  ожидаемый тип значения
     * @param <T>   тип результата
     * @return значение из кэша или {@code null}, если прочитать не удалось
     */
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

    /**
     * Нормализует входные ключи:
     * <ul>
     *   <li>отбрасывает {@code null};</li>
     *   <li>{@code trim()};</li>
     *   <li>отбрасывает пустые строки после {@code trim};</li>
     *   <li>удаляет дубликаты, сохраняя порядок первого вхождения.</li>
     * </ul>
     *
     * @param keys исходный список ключей
     * @return нормализованный список ключей
     */
    @NonNull
    private List<String> normalize(@NonNull List<String> keys) {
        return keys.stream()
                .filter(Objects::nonNull)
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .distinct()
                .collect(Collectors.toCollection(ArrayList::new));
    }

    /**
     * Возвращает первый элемент списка или {@code null}, если список пустой или равен {@code null}.
     *
     * @param list список
     * @param <T>  тип элемента
     * @return первый элемент или {@code null}
     */
    @Nullable
    private static <T> T safeFirst(@Nullable List<T> list) {
        return (list == null || list.isEmpty()) ? null : list.get(0);
    }
}

```
