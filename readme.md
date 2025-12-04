```java

@NonNull
    public List<CurrencyDto> getAllCurrencies() {
        return currencyCacheOps.loadAllCurrencies();
    }

    /** Валюты по списку кодов */
    @NonNull
    public List<CurrencyDto> getCurrenciesByCodes(@NonNull List<String> currencyCodes) {
        if (currencyCodes.isEmpty()) {
            // либо пустой список, либо тут можно бросить BadRequest
            return List.of();
        }
        return cacheGetOrLoadService.fetchData(CURRENCY_BY_CODE, currencyCodes);
    }

```
