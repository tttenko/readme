```java
/** Получает полный список территориальных банков. */
    @GetMapping("/tb/all")
    @Operation(
        operationId = "getAllTerBanks",
        summary = "Получение списка всех территориальных банков",
        description = """
            Возвращает полный список территориальных банков на основе данных из сервиса ЦС МД.
            Используется для справочников/выпадающих списков.
            """
    )
    ResultObj<List<TerBankDto>> getAllTerBanks();

    /** Получает список территориальных банков по заданному списку кодов. */
    @GetMapping("/tb")
    @Operation(
        operationId = "getTerBank",
        summary = "Получение списка территориальных банков по коду(ам)",
        description = """
            Возвращает список территориальных банков по параметру tbCode (повторяющийся параметр в query).
            tbCode обязателен и должен содержать хотя бы одно значение.
            Для получения полного списка используйте /tb/all.
            """
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Успешный поиск территориальных банков"),
        @ApiResponse(responseCode = "400", description = "Некорректные параметры запроса (валидация, пустой/отсутствующий tbCode)"),
        @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<TerBankDto>> getTerBank(
        @RequestParam(name = "tbCode")
        @NotEmpty(message = "Параметр tbCode должен содержать хотя бы одно значение")
        List<@NotBlank(message = "Код территориального банка не должен быть пустым") String> tbCodes
    );

    /** Получает информацию о территориальном банке по его коду. */
    @GetMapping("/tb/{tbCode}")
    @Operation(
        operationId = "getTerBankById",
        summary = "Получение территориального банка по коду",
        description = """
            Возвращает территориальный банк на основе данных из сервиса ЦС МД по коду (tbCode).
            Если по коду ничего не найдено — возвращается ошибка 404.
            """
    )
    ResultObj<List<TerBankDto>> getTerBankById(
        @PathVariable("tbCode")
        @NotBlank(message = "tbCode не должен быть пустым")
        String tbCode
    );

    /** Получает список всех территориальных банков с реквизитами. */
    @GetMapping("/tb-requisite/all")
    @Operation(
        operationId = "getAllTerBanksRequisite",
        summary = "Получение списка всех территориальных банков с реквизитами",
        description = """
            Возвращает полный список территориальных банков вместе с реквизитами
            на основе данных из сервиса ЦС МД.
            """
    )
    ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteALL();

    /** Получает список ТБ с реквизитами по заданному списку кодов. */
    @GetMapping("/tb-requisite")
    @Operation(
        operationId = "getTerBanksRequisite",
        summary = "Получение списка ТБ с реквизитами по коду(ам)",
        description = """
            Возвращает список территориальных банков с реквизитами по параметру tbCode (повторяющийся параметр в query).
            tbCode обязателен и должен содержать хотя бы одно значение.
            Для получения полного списка используйте /tb-requisite/all.
            """
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Успешный поиск ТБ с реквизитами"),
        @ApiResponse(responseCode = "400", description = "Некорректные параметры запроса (валидация, пустой/отсутствующий tbCode)"),
        @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisite(
        @RequestParam(name = "tbCode")
        @NotEmpty(message = "Параметр tbCode должен содержать хотя бы одно значение")
        List<@NotBlank(message = "Код территориального банка не должен быть пустым") String> tbCodes
    );

    /** Получает территориальный банк с реквизитами по коду. */
    @GetMapping("/tb-requisite/{tbCode}")
    @Operation(
        operationId = "getTerBanksWithRequisiteByCodes",
        summary = "Получение ТБ с реквизитами по коду",
        description = """
            Возвращает территориальный банк с реквизитами по коду (tbCode).
            Если по коду ничего не найдено — возвращается ошибка 404.
            """
    )
    ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteById(
        @PathVariable("tbCode")
        @NotBlank(message = "tbCode не должен быть пустым")
        String tbCode
    );
```
