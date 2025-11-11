```java

public interface BatchLoader<T> {
  /** Имя кэша, с которым работает загрузчик (должно совпадать с cacheName в @Cacheable/константах). */
  String cacheName();

  /** Тип элементов в кэше. */
  Class<T> type();

  /** Как получить ключ из объекта (для записи в кэш и восстановления порядка). */
  String keyOf(T value);

  /** Грузим промахи одним запросом. */
  List<T> loadByKeys(List<String> keys);

  /* Безопасная обёртка (вынесли из BatchCacheSupport). */
  default List<T> loadSafely(List<String> keys) {
    try {
      var res = loadByKeys(keys);
      return res == null ? List.of() : res;
    } catch (RuntimeException ex) {
      return List.of();
    }
  }
}
Пример реализаций (микро-классы, по 10–20 строк):

java
Копировать код
@Component
class TerBankByCodeLoader implements BatchLoader<TerBankDto> {
  private final BaseMasterDataRequestService master;
  private final TerBankMapper mapper;
  private final SearchRequestProperties props;

  // конструктор через @RequiredArgsConstructor

  @Override public String cacheName() { return TerBankService2.TB_BY_CODE; }
  @Override public Class<TerBankDto> type() { return TerBankDto.class; }
  @Override public String keyOf(TerBankDto v) { return v.getTbCode(); }

  @Override public List<TerBankDto> loadByKeys(List<String> codes) {
    var resp = master.requestData(props.getSlugValueForTerBank(), codes, SearchRequestProperties.Context.BOOK);
    return createResult(resp, mapper);
  }
}

@Component
class TerBankReqByCodeLoader implements BatchLoader<TerBankWithRequisiteDto> {
  private final BaseMasterDataRequestService master;
  private final TerBankWithRequisiteMapper mapper;
  private final SearchRequestProperties props;

  @Override public String cacheName() { return TerBankService2.TB_REQ_BY_CODE; }
  @Override public Class<TerBankWithRequisiteDto> type() { return TerBankWithRequisiteDto.class; }
  @Override public String keyOf(TerBankWithRequisiteDto v) { return v.getTbCode(); }

  @Override public List<TerBankWithRequisiteDto> loadByKeys(List<String> codes) {
    var resp = master.requestDataWithAttribute(props.getSlugValueForTerBank(), codes, SearchRequestProperties.Context.BOOK);
    return createWithAttribute(resp, mapper);
  }
}
2) Упрощённый BatchCacheSupport (только кэш + оркестрация)
java
Копировать код
@Slf4j
@Component
public class BatchCacheSupport {
  private final CacheManager cacheManager;
  private final Map<String, BatchLoader<?>> loadersByCache;

  public BatchCacheSupport(CacheManager cacheManager, List<BatchLoader<?>> loaders) {
    this.cacheManager = cacheManager;
    this.loadersByCache = loaders.stream()
        .collect(Collectors.toMap(BatchLoader::cacheName, Function.identity()));
  }

  @SuppressWarnings("unchecked")
  @NonNull
  public <T> List<T> fetchBatch(@NonNull String cacheName, @NonNull List<String> keys) {
    if (keys.isEmpty()) return List.of();

    BatchLoader<T> loader = (BatchLoader<T>) loadersByCache.get(cacheName);
    if (loader == null) throw new IllegalArgumentException("No BatchLoader for cache " + cacheName);

    Cache cache = cacheManager.getCache(cacheName);

    Map<String, T> hits = new LinkedHashMap<>();
    List<String> miss = new ArrayList<>();

    collectHitsAndMisses(cache, keys, loader.type(), hits, miss);

    List<T> loaded = miss.isEmpty() ? List.of() : loader.loadSafely(miss);

    putLoadedToCache(cache, loaded, loader::keyOf);

    return orderByOriginal(keys, hits, loaded, loader::keyOf);
  }

  /* дальше — ваши же утилиты без изменений: collectHitsAndMisses, putLoadedToCache, orderByOriginal */
}
```
