```java
@RequestMapping(value = "/api/v1", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
@Tag(
    name = "Supplier controller",
    description = "Предоставление информации о профиле Поставщика из АС Мастер-данные"
)
@SecurityRequirement(name = "Authorization")
public interface UiSupplierController {

  /**
   * Получает информацию о контрагентах по ИНН и опционально по КПП.
   */
  @GetMapping(value = "/supplier-search")
  @Operation(
      operationId = "getCounterpartyRequisites",
      summary = "Предоставление информации о получении контрагента по ИНН/КПП",
      description =
          "Возвращает список контрагентов, соответствующих критериям поиска. "
              + "Поиск выполняется по обязательному параметру `inn` и опциональному `kpp`."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешный поиск контрагентов",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректные параметры запроса (валидация inn/kpp)",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "500",
          description = "Внутренняя ошибка сервиса",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      )
  })
  ResultObj<List<CounterpartyDto>> searchSupplier(
      @RequestParam(name = "inn")
      @Parameter(
          name = "inn",
          in = ParameterIn.QUERY,
          required = true,
          description = "ИНН контрагента (обязательный параметр)",
          schema = @Schema(type = "string", minLength = 10, maxLength = 12, example = "7707083893")
      )
      String inn,

      @RequestParam(required = false, name = "kpp")
      @Parameter(
          name = "kpp",
          in = ParameterIn.QUERY,
          required = false,
          description = "КПП контрагента (опционально)",
          schema = @Schema(type = "string", minLength = 9, maxLength = 9, example = "773601001")
      )
      String kpp
  );

  /**
   * Получает информацию о контрагентах по списку идентификаторов.
   * Если список пустой — возвращаются все доступные контрагенты.
   */
  @GetMapping(value = "/supplier")
  @Operation(
      operationId = "getSupplier",
      summary = "Предоставление информации о получении контрагентов по списку идентификаторов",
      description =
          "Возвращает список контрагентов по переданному списку идентификаторов `id`. "
              + "Если параметр `id` не задан или список пустой — возвращаются все доступные контрагенты. "
              + "Для передачи нескольких значений повторяйте query-параметр: `?id=1&id=2&id=3`."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение списка контрагентов",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректные параметры запроса (валидация id)",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "500",
          description = "Внутренняя ошибка сервиса",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      )
  })
  ResultObj<List<CounterpartyDto>> getSupplier(
      @RequestParam(required = false, name = "id")
      @ArraySchema(
          schema = @Schema(
              title = "Идентификатор контрагента",
              description = "Идентификатор контрагента",
              type = "string",
              maxLength = 255,
              example = "12345"
          ),
          maxItems = 999
      )
      @Parameter(
          name = "id",
          in = ParameterIn.QUERY,
          required = false,
          description =
              "Список идентификаторов контрагентов. "
                  + "Можно передать несколько значений, повторяя query-параметр: `?id=1&id=2`. "
                  + "Если параметр не задан — возвращаются все контрагенты."
      )
      List<String> listOfId
  );

  /**
   * Получает информацию о контрагенте по его идентификатору.
   * Если контрагент не найден — возвращается ошибка 404.
   */
  @GetMapping(value = "/supplier/{id}")
  @Operation(
      operationId = "getSupplierById",
      summary = "Предоставление информации о получении контрагента по идентификатору",
      description =
          "Возвращает информацию о контрагенте по его идентификатору `id`. "
              + "Если контрагент не найден — возвращается ошибка 404 (MdaDataNotFoundException)."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение информации о контрагенте",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректный идентификатор контрагента (валидация параметра)",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "404",
          description = "Контрагент не найден",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "500",
          description = "Внутренняя ошибка сервиса",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      )
  })
  ResultObj<List<CounterpartyDto>> getSupplierById(
      @PathVariable("id")
      @Parameter(
          name = "id",
          in = ParameterIn.PATH,
          required = true,
          description = "Идентификатор контрагента",
          schema = @Schema(type = "string", maxLength = 255, example = "12345")
      )
      String id
  );

  /**
   * Получает банковские реквизиты по списку идентификаторов поставщиков.
   */
  @GetMapping(value = "/supplier-bank-requisite")
  @Operation(
      operationId = "getSupplierRequisite",
      summary = "Получения банковских реквизитов по списку идентификаторов поставщиков",
      description =
          "Возвращает банковские реквизиты поставщиков по списку идентификаторов `id`. "
              + "Для передачи нескольких значений повторяйте query-параметр: `?id=1&id=2&id=3`."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение банковских реквизитов",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректные параметры запроса (валидация id)",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "500",
          description = "Внутренняя ошибка сервиса",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      )
  })
  ResultObj<List<BankDto>> getSupplierRequisite(
      @RequestParam(name = "id")
      @ArraySchema(
          schema = @Schema(
              title = "Идентификатор поставщика",
              description = "Идентификатор поставщика",
              type = "string",
              maxLength = 255,
              example = "12345"
          ),
          maxItems = 999
      )
      @Parameter(
          name = "id",
          in = ParameterIn.QUERY,
          required = true,
          description =
              "Список идентификаторов поставщиков. "
                  + "Можно передать несколько значений, повторяя query-параметр: `?id=1&id=2`."
      )
      List<String> listOfId
  );

  /**
   * Получает банковские реквизиты по идентификатору поставщика.
   * Если реквизиты не найдены — возвращается ошибка 404.
   */
  @GetMapping(value = "/supplier-bank-requisite/{id}")
  @Operation(
      operationId = "getSupplierRequisiteById",
      summary = "Получения банковских реквизитов по идентификатору поставщика",
      description =
          "Возвращает банковские реквизиты поставщика по идентификатору `id`. "
              + "Если реквизиты не найдены — возвращается ошибка 404 (MdaDataNotFoundException)."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение банковских реквизитов",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректный идентификатор поставщика (валидация параметра)",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "404",
          description = "Банковские реквизиты не найдены",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "500",
          description = "Внутренняя ошибка сервиса",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      )
  })
  ResultObj<List<BankDto>> getSupplierRequisiteById(
      @PathVariable("id")
      @Parameter(
          name = "id",
          in = ParameterIn.PATH,
          required = true,
          description = "Идентификатор поставщика",
          schema = @Schema(type = "string", maxLength = 255, example = "12345")
      )
      String id
  );
}
```
