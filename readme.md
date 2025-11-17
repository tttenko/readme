```java

@Service
@RequiredArgsConstructor
public class CurrencyCacheOps {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final CurrencyMapper currencyMapper;

    /**
     * Грузит все валюты и кэширует результат под ключом 'ALL'.
     */
    @Cacheable(cacheNames = CurrencyService2.CURRENCY_ALL, key = "'ALL'", sync = true)
    @NonNull
    public List<CurrencyDto> loadAllCurrencies() {
        GetItemsSearchResponse resp =
                baseMasterDataRequestService.requestDataByAttributes(
                        properties.getSlugValueForCurrency(),
                        properties.getCurrencyAttributeId(),
                        null // searchFieldValue
                );

        return BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper);
    }
}

@Component
@RequiredArgsConstructor
public class LoaderCurrencyByCode implements BatchLoader<CurrencyDto> {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final CurrencyMapper currencyMapper;

    @Override
    public String cacheName() {
        return CurrencyService2.CURRENCY_BY_CODE;
    }

    @Override
    public Class<CurrencyDto> elementType() {
        return CurrencyDto.class;
    }

    @Override
    public String extractKey(CurrencyDto value) {
        return value.getCurrencyCode();
    }

    @Override
    @NonNull
    public List<CurrencyDto> fetchByKeys(@NonNull List<String> keys) {
        if (keys.isEmpty()) {
            return List.of();
        }

        GetItemsSearchResponse resp =
                baseMasterDataRequestService.requestDataByAttributes(
                        properties.getSlugValueForCurrency(),
                        properties.getCurrencyAttributeId(),
                        keys
                );

        return BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper);
    }
}

@Slf4j
@Service
@RequiredArgsConstructor
public class CurrencyService2 {

    public static final String CURRENCY_BY_CODE = "currency_by_code";
    public static final String CURRENCY_ALL     = "currency_all";

    private final CurrencyCacheOps currencyCacheOps;
    private final CacheGetOrLoadService cacheGetOrLoadService;

    @NonNull
    public ResultObj<List<CurrencyDto>> searchCurrenciesByCode(
            @Nullable final List<String> currencyCodes
    ) {
        boolean requestAll = (currencyCodes == null || currencyCodes.isEmpty());

        List<CurrencyDto> data = requestAll
                ? currencyCacheOps.loadAllCurrencies()
                : cacheGetOrLoadService.fetchData(CURRENCY_BY_CODE, currencyCodes);

        return getSuccessResponse(data);
    }
}

```
