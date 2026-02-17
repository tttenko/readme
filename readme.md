```java
@GetMapping("/currencyRate")
@Operation(operationId = "getCurrencyRate",
        summary = "Получение курса по параметрам (fromCurrencyIsoNum, toCurrencyIsoNum, date)")
ResultObj<List<CurrencyRateDto>> getCurrencyRate(

        @Schema(description = "Дата в формате dd.MM.yyyy", example = "11.02.2026")
        @RequestParam("date")
        @NotNull
        @DateTimeFormat(pattern = "dd.MM.yyyy")
        LocalDate date,

        @Schema(description = "ISO numeric код исходной (base) валюты. Соответствует FXRates@ISOnum1", example = "036")
        @RequestParam("fromCurrencyIsoNum")
        @NotBlank
        String fromCurrencyIsoNum,

        @Schema(description = "ISO numeric код целевой (quote) валюты. Соответствует FXRates@ISOnum2", example = "643")
        @RequestParam("toCurrencyIsoNum")
        @NotBlank
        String toCurrencyIsoNum
);

@Override
public ResultObj<List<CurrencyRateDto>> getCurrencyRate(
        @RequestParam("date") LocalDate date,
        @RequestParam("fromCurrencyIsoNum") String fromCurrencyIsoNum,
        @RequestParam("toCurrencyIsoNum") String toCurrencyIsoNum) {

    List<CurrencyRateDto> items = currencyService.getCurrencyRate(date, fromCurrencyIsoNum, toCurrencyIsoNum);

    String description = "Найдено записей: %d на дату %s для пересчета из %s в %s"
            .formatted(items.size(), date, fromCurrencyIsoNum, toCurrencyIsoNum);

    ResultObj<List<CurrencyRateDto>> result = new ResultObj<>();
    result.addMessages(success(message: "Поиск выполнен", target: null, description));
    result.setData(items);
    result.setCount((long) items.size());
    return result;
}
```
