```java

@Slf4j
@Service
@RequiredArgsConstructor
public class CurrencyService {

  public static final String CURRENCY_BY_CODE = "currency_by_code";
  public static final String CURRENCY_ALL = "currency_all";

  private final BatchCacheSupport batchLoad;
  private final CurrencyCacheOps currencyCacheOps;

  /**
   * Возвращает список валют по коду(ам). Если коды не заданы — вернёт все валюты.
   * Для списка кодов выполняется батч-чтение/догрузка miss одним походом в МД.
   */
  @Nonnull
  public ResultObj<List<CurrencyDto>> searchCurrenciesByCode(@Nullable final List<String> currencyCodes) {

    // нормализуем и дедуплицируем вход
    final List<String> keys = (currencyCodes == null) ? List.of() :
        currencyCodes.stream()
            .filter(Objects::nonNull)
            .map(String::trim)
            .filter(s -> !s.isEmpty())
            .distinct()
            .toList();

    final List<CurrencyDto> data = CollectionUtils.isEmpty(keys)
        // кейс "все валюты" — отдельный кэш одной записи
        ? currencyCacheOps.getAll()
        // батч-кэш по кодам (один запрос на miss, раскладка по ключам через extractor)
        : batchLoad.fetchBatch(
            CURRENCY_BY_CODE,
            keys,
            currencyCacheOps::loadByCodes,
            CurrencyDto::getCurrencyCode,
            CurrencyDto.class
          );

    return getSuccessResponse(data);
  }
}

@Service
@RequiredArgsConstructor
public class CurrencyCacheOps {

  private final BaseMasterDataRequestService baseMasterDataRequestService;
  private final SearchRequestProperties properties;
  private final CurrencyMapper currencyMapper;

  /** Кэш "всё", одна запись, thread-safe загрузка. */
  @Cacheable(cacheNames = CurrencyService.CURRENCY_ALL, key = "'ALL'", sync = true)
  @Nonnull
  public List<CurrencyDto> getAll() {
    final GetItemsSearchResponse resp = baseMasterDataRequestService.requestDataByAttributes(
        properties.getSlugValueForCurrency(),
        properties.getCurrencyAttributeId(),
        null // null/empty → "все" (как в старой реализации)
    );
    return createResultWithAttribute(resp, currencyMapper);
  }

  /** Батч-лоадер для miss-ключей по кодам (без аннотаций кеша). */
  @Nonnull
  public List<CurrencyDto> loadByCodes(@Nonnull final List<String> codes) {
    final GetItemsSearchResponse resp = baseMasterDataRequestService.requestDataByAttributes(
        properties.getSlugValueForCurrency(),
        properties.getCurrencyAttributeId(),
        codes
    );
    return createResultWithAttribute(resp, currencyMapper);
  }
}


```
