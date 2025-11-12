```java

@Slf4j
@Component
@RequiredArgsConstructor
public class BatchCacheSupport {

  private final CacheManager cacheManager;

  /** Возвращает найденные в кэше элементы по ключам. */
  public <T> Map<String, T> collectHits(
      @NonNull String cacheName,
      @NonNull List<String> keys,
      @NonNull Class<T> type
  ) {
    Map<String, T> hits = new LinkedHashMap<>();
    if (keys.isEmpty()) return hits;

    Cache cache = cacheManager.getCache(cacheName);
    if (cache == null) return hits;

    for (String key : keys) {
      if (key == null || key.isBlank()) {
        log.debug("Skip invalid cache key: '{}'", key);
        continue;
      }
      T v = cache.get(key, type);
      if (v != null) hits.put(key, v);
    }
    return hits;
  }

  /** Возвращает промахи: валидные ключи, которых нет среди hits. */
  public List<String> collectMisses(
      @NonNull List<String> keys,
      @NonNull Map<String, ?> hits
  ) {
    List<String> miss = new ArrayList<>();
    for (String key : keys) {
      if (key == null || key.isBlank()) continue;
      if (!hits.containsKey(key)) miss.add(key);
    }
    return miss;
  }

  /** Кладёт загруженные элементы в кэш. Некорректные ключи пропускает. */
  public <T> void putLoadedToCache(
      @NonNull String cacheName,
      @NonNull List<T> loaded,
      @NonNull Function<T, String> keyExtractor
  ) {
    if (loaded.isEmpty()) return;
    Cache cache = cacheManager.getCache(cacheName);
    if (cache == null) return;

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

  /** Собирает ответ в порядке входных ключей. */
  public <T> List<T> orderByOriginal(
      @NonNull List<String> originalKeys,
      @NonNull Map<String, T> hits,
      @NonNull List<T> loaded,
      @NonNull Function<T, String> keyExtractor
  ) {
    Map<String, T> byKey = new HashMap<>(hits);
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

@Service
@RequiredArgsConstructor
public class BatchCacheFacade {
  private final BatchCacheSupport cache;
  private final Map<String, BatchLoader<?>> loaders; // имя бина == cacheName

  @SuppressWarnings("unchecked")
  public <T> List<T> fetch(@NonNull String cacheName, @NonNull List<String> keys) {
    if (keys.isEmpty()) return List.of();

    BatchLoader<T> loader = (BatchLoader<T>) loaders.get(cacheName);
    if (loader == null) throw new IllegalArgumentException("No loader for " + cacheName);

    Map<String, T> hits = cache.collectHits(cacheName, keys, loader.type());
    List<String> miss = cache.collectMisses(keys, hits);               // без стат. класса
    // по желанию: miss = miss.stream().distinct().toList();
    List<T> loaded = miss.isEmpty() ? List.of() : loader.loadSafely(miss);

    cache.putLoadedToCache(cacheName, loaded, loader::keyOf);
    return cache.orderByOriginal(keys, hits, loaded, loader::keyOf);
  }
}

/**
 * Фасад батч-доступа к кэшу.
 *
 * Инкапсулирует типовой алгоритм:
 * 1) ищет значения по ключам в кэше (hits);
 * 2) недостающие ключи (misses) загружает через соответствующий {@link BatchLoader};
 * 3) складывает загруженные значения в кэш;
 * 4) возвращает результат в порядке входных ключей.
 *
 * Реестр лоадеров формируется из всех Spring-бинов {@code BatchLoader<?>}
 * и адресуется по {@code cacheName}. Сервисы работают только с этим фасадом,
 * не зная о деталях кэширования и загрузки.
 */

 /**
 * Регистрирует все доступные {@link BatchLoader} по их {@link BatchLoader#cacheName()}.
 * Вызывается контейнером после инъекции зависимостей.
 * Требование: имена кэшей должны быть уникальны.
 */

 /**
 * Получает значения по ключам из кэша с догрузкой «промахов».
 *
 * @param cacheName имя кэша; должно совпадать с {@link BatchLoader#cacheName()} зарегистрированного лоадера
 * @param keys      список ключей; пустые/blank/null ключи игнорируются
 * @param <T>       тип значений, обслуживаемых лоадером для данного кэша
 * @return неизменяемый список значений в порядке входных ключей; для не найденных ключей элементы отсутствуют
 * @throws IllegalArgumentException если лоадер для указанного кэша не зарегистрирован
 * @throws MdaBatchLoadException    если лоадер вернул null или выбросил исключение
 */
```
