```java
@RequestMapping(value = "/api/v1/info/prod_calend_date", produces = MediaType.APPLICATION_JSON_VALUE)
@Tag(
    name = "Prod calendar date controller",
    description = "Получение информации о дате: рабочий или нерабочий день (по производственному календарю)"
)
public interface ProdCalendDateController {

    String DATE_REGEX = "^\\d{2}\\.\\d{2}\\.\\d{4}$";

    /**
     * Поиск по массиву дат.
     * Если даты не переданы — используется текущая дата по Москве.
     */
    @GetMapping
    @Operation(
        operationId = "searchProdCalendDate",
        summary = "Получение информации о дате (рабочий/нерабочий) по массиву дат"
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Успешный поиск (в т.ч. пустой результат)"),
        @ApiResponse(responseCode = "400", description = "Некорректные параметры запроса (валидация)"),
        @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<ProdCalendDateDto>> searchProdCalendDates(
        @RequestParam(name = "date", required = false)
        List<
            @NotBlank(message = "Параметр date не должен быть пустым")
            @Pattern(regexp = DATE_REGEX, message = "Дата должна быть в формате dd.MM.yyyy")
            String
        > dates
    );

    /**
     * Поиск одной даты (как в спецификации GET .../{date}).
     */
    @GetMapping("/{date}")
    @Operation(
        operationId = "searchProdCalendDateByDate",
        summary = "Получение информации о дате (рабочий/нерабочий) по одной дате"
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Успешный поиск (в т.ч. пустой результат)"),
        @ApiResponse(responseCode = "400", description = "Некорректный формат даты"),
        @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<ProdCalendDateDto>> searchProdCalendDateByDate(
        @PathVariable("date")
        @NotBlank(message = "date не должен быть пустым")
        @Pattern(regexp = DATE_REGEX, message = "Дата должна быть в формате dd.MM.yyyy")
        String date
    );
}
```
