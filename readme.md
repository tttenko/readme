```java
 /** Полный список валют */
    @GetMapping("/all")
    @Operation(
        operationId = "getAllCurrencies",
        summary = "Предоставление информации обо всех доступных валютах"
    )
    ResultObj<List<CurrencyDto>> getAllCurrencies();

    /** Поиск валют по списку кодов */
    @GetMapping
    @Operation(
        operationId = "searchCurrenciesByCodes",
        summary = "Предоставление информации о валютах по списку кодов",
        description = """
            Возвращает список валют по параметру currencyCode (повторяющийся параметр в query).
            currencyCode обязателен и должен содержать хотя бы одно значение.
            Для получения полного списка используйте /currency/all.
            """
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Успешный поиск валют"),
        @ApiResponse(responseCode = "400", description = "Некорректные параметры запроса (валидация, пустой/отсутствующий currencyCode)"),
        @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<CurrencyDto>> searchCurrenciesByCodes(
        @RequestParam(name = "currencyCode")
        @NotEmpty(message = "Параметр currencyCode должен содержать хотя бы одно значение")
        List<@NotBlank(message = "Код валюты не должен быть пустым") String> currencyCodes
    );

    /** Валюта по одному коду */
    @GetMapping("/{currencyCode}")
    @Operation(
        operationId = "searchCurrencyByCode",
        summary = "Предоставление информации о валюте по коду"
    )
    ResultObj<List<CurrencyDto>> searchCurrencyByCode(
        @PathVariable("currencyCode")
        @NotBlank(message = "currencyCode не должен быть пустым")
        String currencyCode
    );
```
