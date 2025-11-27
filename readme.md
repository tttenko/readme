```java

@Operation(
        operationId = "searchCountry",
        summary = "Получение перечня стран по массиву кодов ALPHA-2"
)
@ApiResponses({
        @ApiResponse(
                responseCode = "200",
                description = "Успешный поиск стран"
        ),
        @ApiResponse(
                responseCode = "400",
                description = "Некорректные параметры запроса (валидация, пустой/отсутствующий countryCode)",
                content = @Content(
                        mediaType = "application/json"
                        // при желании можно указать схему ошибки:
                        // schema = @Schema(implementation = MessageObj.class)
                )
        ),
        @ApiResponse(
                responseCode = "500",
                description = "Внутренняя ошибка сервиса",
                content = @Content(mediaType = "application/json")
        )
})

@Operation(
        operationId = "searchCountryByCode",
        summary = "Предоставление информации о стране по коду ALPHA-2"
)
@ApiResponses({
        @ApiResponse(responseCode = "200", description = "Успешное получение информации о стране"),
        @ApiResponse(
                responseCode = "400",
                description = "Некорректный код страны (валидация параметра)",
                content = @Content(mediaType = "application/json")
        ),
        @ApiResponse(
                responseCode = "500",
                description = "Внутренняя ошибка сервиса",
                content = @Content(mediaType = "application/json")
        )
})

@Operation(
        operationId = "getAllCountries",
        summary = "Получение полного перечня стран"
)
@ApiResponses({
        @ApiResponse(responseCode = "200", description = "Успешное получение списка стран"),
        @ApiResponse(
                responseCode = "500",
                description = "Внутренняя ошибка сервиса",
                content = @Content(mediaType = "application/json")
        )
})
```
