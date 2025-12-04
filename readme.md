```java

// Все валюты
@GetMapping(value = "/all", produces = APPLICATION_JSON_UTF8_VALUE)
@Operation(
        operationId = "getAllCurrencies",
        summary = "Предоставление информации обо всех доступных валютах"
)
public ResultObj<List<CurrencyDto>> getAllCurrencies() {
    return getSuccessResponse(currencyService.getAllCurrencies());
}

// Валюты по списку кодов
@GetMapping(produces = APPLICATION_JSON_UTF8_VALUE)
@Operation(
        operationId = "searchCurrenciesByCodes",
        summary = "Предоставление информации о валютах по списку кодов"
)
public ResultObj<List<CurrencyDto>> searchCurrenciesByCodes(
        @RequestParam(name = "currencyCode") List<String> currencyCodes) {

    return getSuccessResponse(currencyService.getCurrenciesByCodes(currencyCodes));
}

```
