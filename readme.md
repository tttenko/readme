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
 * <p><b>BatchCacheFacade</b> — фасад для унифицированного батч-доступа к кэшу.</p>
 *
 * <p>Инкапсулирует общий алгоритм работы с данными:
 * <ul>
 *   <li>1️⃣ Получает значения по ключам из кэша (<b>hits</b>);</li>
 *   <li>2️⃣ Определяет отсутствующие ключи (<b>misses</b>);</li>
 *   <li>3️⃣ Загружает отсутствующие данные через {@link BatchLoader};</li>
 *   <li>4️⃣ Кэширует загруженные значения;</li>
 *   <li>5️⃣ Возвращает результат в порядке исходных ключей.</li>
 * </ul>
 * </p>
 *
 * <p>Реестр загрузчиков формируется автоматически из всех Spring-бинов,
 * реализующих {@link BatchLoader}, и индексируется по значению {@link BatchLoader#cacheName()}.</p>
 *
 * <p>Сервисы используют только этот фасад — не взаимодействуют напрямую с {@link BatchCacheSupport}
 * и не содержат логики кэширования или загрузки данных.</p>
 *
 * <h3>Пример использования:</h3>
 * <pre>{@code
 * List<TerBankDto> banks = batchCacheFacade.fetch(TerBankService2.TB_BY_CODE, tbCodes);
 * }</pre>
 *
 * @see BatchCacheSupport
 * @see BatchLoader
 * @author tttenko
 */

 /**
 * Инициализирует реестр доступных {@link BatchLoader}-ов после внедрения зависимостей.
 * <p>Ключом является {@link BatchLoader#cacheName()}.</p>
 * <p>Требование: имена кэшей должны быть уникальны.</p>
 */

/**
 * Возвращает данные по ключам из кэша с автоматической догрузкой «промахов».
 *
 * @param cacheName имя кэша; должно совпадать с {@link BatchLoader#cacheName()}
 * @param keys список ключей; null/пустые/blank ключи игнорируются
 * @param <T> тип данных, обслуживаемых данным кэшем
 * @return список элементов в порядке исходных ключей (только найденные и успешно загруженные)
 *
 * @throws IllegalArgumentException если не найден {@link BatchLoader} для указанного кэша
 * @throws MdaBatchLoadException если загрузчик вернул null или произошла ошибка загрузки
 */
```
