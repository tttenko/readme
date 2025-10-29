```java

@Slf4j
@Service
@RequiredArgsConstructor
public class SupplierService {

  public static final String SUPPLIER_REQ_BY_SUPPLIER_ID = "supplier_req_by_supplier_id";
  public static final String SUPPLIER_BY_ID = "supplier_by_id";
  public static final String SUPPLIER_BY_INN_KPP = "supplier_by_inn_kpp";

  private final BatchCacheSupport batchLoad;
  private final SupplierRequisiteMapper supplierRequisiteMapper;
  private final SupplierMapper supplierMapper;

  private final BaseMasterDataRequestService base;
  private final SearchRequestProperties properties;

  @Nonnull
  public ResultObj<List<BankDto>> searchSupplierRequisite(@Nonnull final List<String> ids) {
    final var list = batchLoad.fetchBatch(
        SUPPLIER_REQ_BY_SUPPLIER_ID,
        ids,
        this::loadRequisitesBySupplierIds,
        BankDto::getSupplierId,
        BankDto.class
    );
    return getSuccessResponse(list);
  }

  @Nonnull
  public ResultObj<List<CounterpartyDto>> searchCounterpartiesByCriteria(@Nullable final String inn,
                                                                         @Nullable final String kpp) {
    final var criteria = buildCriteriaMap(inn, kpp);
    if (criteria.size() != 2) {
      return getSupplierByCriteria(criteria);
    }
    final String key = buildInnKppKey(inn, kpp);
    final var list = batchLoad.fetchBatch(
        SUPPLIER_BY_INN_KPP,
        List.of(key),
        missed -> loadSuppliersByCriteria(criteria),
        dto -> buildInnKppKey(dto.getInn(), dto.getKpp()),
        CounterpartyDto.class
    );
    return getSuccessResponse(list);
  }

  @Nonnull
  public ResultObj<List<CounterpartyDto>> getCounterpartiesById(@Nonnull final List<String> ids) {
    final var list = batchLoad.fetchBatch(
        SUPPLIER_BY_ID,
        ids,
        this::loadSuppliersByIds,
        CounterpartyDto::getId,
        CounterpartyDto.class
    );
    return getSuccessResponse(list);
  }

  // -------- Загрузчики из мастер-данных --------

  @Nonnull
  private List<BankDto> loadRequisitesBySupplierIds(@Nonnull final List<String> ids) {
    final GetItemsSearchResponse resp = base.requestDataWithRefItemSlug2(
        properties.getSlugValueForSupplierRequisite(),
        properties.getAttributeIdForSupplierRequisite(),
        ids
    );
    return createWithAttribute(resp, supplierRequisiteMapper);
  }

  @Nonnull
  private List<CounterpartyDto> loadSuppliersByIds(@Nonnull final List<String> ids) {
    final GetItemsSearchResponse resp = base.requestDataWithAttribute(
        properties.getSlugValueForCounterparty(),
        ids,
        SearchRequestProperties.Context.BOOK
    );
    return createWithAttribute(resp, supplierMapper);
  }

  @Nonnull
  private List<CounterpartyDto> loadSuppliersByCriteria(@Nonnull final Map<String, List<String>> criteria) {
    final GetItemsSearchResponse resp = base.requestDataWithAttribute(
        properties.getSlugValueForCounterparty(),
        criteria
    );
    return createWithAttribute(resp, supplierMapper);
  }

  @Nonnull
  private ResultObj<List<CounterpartyDto>> getSupplierByCriteria(@Nonnull final Map<String, List<String>> criteria) {
    return createResultObjWithAttribute(
        base.requestDataWithAttribute(properties.getSlugValueForCounterparty(), criteria),
        supplierMapper
    );
  }

  // -------- Утилиты --------

  @Nonnull
  private Map<String, List<String>> buildCriteriaMap(@Nullable final String inn, @Nullable final String kpp) {
    final Map<String, List<String>> criteria = new HashMap<>(2);
    if (inn != null && !inn.isBlank()) {
      criteria.put(properties.getAttributeForInn(), List.of(inn));
    }
    if (kpp != null && !kpp.isBlank()) {
      criteria.put(properties.getAttributeForKpp(), List.of(kpp));
    }
    return criteria;
  }

  @Nonnull
  private static String buildInnKppKey(@Nullable final String inn, @Nullable final String kpp) {
    return String.format("inn:%s:kpp:%s", String.valueOf(inn), String.valueOf(kpp));
  }
}



/**
 * Вспомогательный сервис для пакетной выборки данных с использованием Spring Cache.
 * Позволяет получать элементы по набору ключей, использовать кэш и догружать промахи батчем.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class BatchCacheSupport {

    private final CacheManager cacheManager;

    /**
     * Получает данные по набору ключей из кэша. Для отсутствующих элементов вызывает загрузчик и обновляет кэш.
     *
     * @param cacheName   имя кэша
     * @param keys        список ключей для выборки
     * @param loader      функция загрузки отсутствующих элементов
     * @param keyExtractor функция для получения ключа из загруженного объекта
     * @param type        тип элементов
     * @param <T>         тип данных
     * @return список элементов в порядке исходных ключей
     */
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

    /**
     * Безопасно вызывает загрузчик, возвращая пустой список при исключениях.
     *
     * @param loader функция загрузки
     * @param miss   список недостающих ключей
     * @param <T>    тип данных
     * @return загруженные элементы или пустой список при ошибке
     */
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

    /**
     * Делит ключи на хиты (нашлись в кэше) и промахи (требуют загрузки).
     *
     * @param cache   кэш
     * @param keys    ключи для выборки
     * @param type    тип элементов
     * @param hitsOut карта найденных элементов
     * @param missOut список отсутствующих ключей
     * @param <T>     тип данных
     */
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
                cached = cache.get(key, type);
            }

            if (cached != null) {
                hitsOut.put(key, cached);
            } else {
                missOut.add(key);
            }
        }
    }

    /**
     * Добавляет загруженные элементы в кэш, пропуская элементы с некорректными ключами.
     *
     * @param cache        кэш для обновления
     * @param loaded       список загруженных элементов
     * @param keyExtractor функция получения ключа из объекта
     * @param <T>          тип данных
     */
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

    /**
     * Собирает итоговый список элементов в порядке исходных ключей.
     *
     * @param originalKeys исходные ключи
     * @param hits         элементы, найденные в кэше
     * @param loaded       загруженные элементы
     * @param keyExtractor функция получения ключа из объекта
     * @param <T>          тип данных
     * @return итоговый список элементов в порядке входных ключей
     */
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
