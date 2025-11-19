```java

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

```
