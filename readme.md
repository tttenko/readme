```java

@Tag(
    name = "TerBanks controller",
    description = "REST API для работы с сервисом АС Мастер-данные (территориальные банки)"
)
@SecurityRequirement(name = "Authorization")
@RequestMapping(
    value = "/ui/v1/info",
    produces = MediaType.APPLICATION_JSON_VALUE
)
public interface UiTerBanksController {

  /**
   * Получает полный список территориальных банков.
   */
  @GetMapping("/tb/all")
  @Operation(
      operationId = "getAllTerBanks",
      summary = "Получение списка всех территориальных банков",
      description =
          "Возвращает полный список территориальных банков на основе данных из сервиса ЦС МД. "
              + "Используется для справочников/выпадающих списков."
  )
  @ApiResponses({
      @ApiResponse(responseCode = "200", description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjTerBankDtoList.class))),
      @ApiResponse(responseCode = "401", description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "403", description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "500", description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)))
  })
  ResultObj<List<TerBankDto>> getAllTerBanks();


  /**
   * Получает список территориальных банков по заданному списку кодов.
   * Если параметр tbCode не указан — возвращается список всех ТБ.
   */
  @GetMapping("/tb")
  @Operation(
      operationId = "getTerBank",
      summary = "Получение списка территориальных банков по коду (или всех, если код не задан)",
      description =
          "Возвращает список территориальных банков на основе данных из сервиса ЦС МД. "
              + "Можно передать список кодов параметром tbCode (повторяя параметр в query). "
              + "Если tbCode не заполнен — возвращаются все территориальные банки."
  )
  @ApiResponses({
      @ApiResponse(responseCode = "200", description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjTerBankDtoList.class))),
      @ApiResponse(responseCode = "400", description = "Запрос не прошёл правила валидации (некорректные параметры)",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "401", description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "403", description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "500", description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)))
  })
  ResultObj<List<TerBankDto>> getTerBank(
      @RequestParam(name = "tbCode", required = false)
      @ArraySchema(
          schema = @Schema(
              description = "Код территориального банка",
              example = "5500",
              type = "string",
              pattern = "^.*$",
              maxLength = 255
          ),
          maxItems = 999
      )
      @Parameter(
          name = "tbCode",
          in = ParameterIn.QUERY,
          description =
              "Список кодов территориальных банков. "
                  + "Можно передать несколько значений, повторяя query-параметр: ?tbCode=5500&tbCode=7700. "
                  + "Если параметр не задан — возвращаются все ТБ.",
          required = false
      )
      List<String> tbCode
  );


  /**
   * Получает информацию о территориальном банке по его коду.
   */
  @GetMapping("/tb/{tbCode}")
  @Operation(
      operationId = "getTerBankById",
      summary = "Получение территориального банка по коду",
      description =
          "Возвращает территориальный банк на основе данных из сервиса ЦС МД по коду (tbCode). "
              + "Если по коду ничего не найдено — возвращается ошибка 404."
  )
  @ApiResponses({
      @ApiResponse(responseCode = "200", description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjTerBankDtoList.class))),
      @ApiResponse(responseCode = "400", description = "Запрос не прошёл правила валидации (некорректный tbCode)",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "401", description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "403", description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "404", description = "Территориальный банк не найден",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "500", description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)))
  })
  ResultObj<List<TerBankDto>> getTerBankById(
      @PathVariable("tbCode")
      @Parameter(
          name = "tbCode",
          in = ParameterIn.PATH,
          description = "Код территориального банка",
          required = true,
          schema = @Schema(
              title = "Код банка",
              description = "Код территориального банка",
              type = "string",
              pattern = "^.*$",
              maxLength = 255,
              example = "5500"
          )
      )
      String tbCode
  );


  /**
   * Получает список всех территориальных банков с реквизитами.
   */
  @GetMapping("/tb-requisite/all")
  @Operation(
      operationId = "getAllTerBanksRequisite",
      summary = "Получение списка всех территориальных банков с реквизитами",
      description =
          "Возвращает полный список территориальных банков вместе с реквизитами на основе данных из сервиса ЦС МД."
  )
  @ApiResponses({
      @ApiResponse(responseCode = "200", description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjTerBankWithRequisiteDtoList.class))),
      @ApiResponse(responseCode = "401", description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "403", description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "500", description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)))
  })
  ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteAll();


  /**
   * Получает список территориальных банков с реквизитами по заданному списку кодов.
   * Если tbCode не передан — возвращается список всех ТБ с реквизитами.
   */
  @GetMapping("/tb-requisite")
  @Operation(
      operationId = "getTerBanksRequisite",
      summary = "Получение списка ТБ с реквизитами по кодам (или всех, если код не задан)",
      description =
          "Возвращает список территориальных банков с реквизитами. "
              + "Можно передать список кодов параметром tbCode (повторяя параметр в query). "
              + "Если tbCode не заполнен — возвращаются все территориальные банки с реквизитами."
  )
  @ApiResponses({
      @ApiResponse(responseCode = "200", description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjTerBankWithRequisiteDtoList.class))),
      @ApiResponse(responseCode = "400", description = "Запрос не прошёл правила валидации (некорректные параметры)",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "401", description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "403", description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "500", description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)))
  })
  ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisite(
      @RequestParam(name = "tbCode", required = false)
      @ArraySchema(
          schema = @Schema(
              description = "Код территориального банка",
              example = "5500",
              type = "string",
              pattern = "^.*$",
              maxLength = 255
          ),
          maxItems = 999
      )
      @Parameter(
          name = "tbCode",
          in = ParameterIn.QUERY,
          description =
              "Список кодов территориальных банков. "
                  + "Можно передать несколько значений, повторяя query-параметр: ?tbCode=5500&tbCode=7700. "
                  + "Если параметр не задан — возвращаются все ТБ с реквизитами.",
          required = false
      )
      List<String> tbCode
  );


  /**
   * Получает территориальный банк с реквизитами по коду.
   */
  @GetMapping("/tb-requisite/{tbCode}")
  @Operation(
      operationId = "getTerBanksWithRequisiteByCodes",
      summary = "Получение ТБ с реквизитами по коду",
      description =
          "Возвращает территориальный банк с реквизитами по коду (tbCode). "
              + "Если по коду ничего не найдено — возвращается ошибка 404."
  )
  @ApiResponses({
      @ApiResponse(responseCode = "200", description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjTerBankWithRequisiteDtoList.class))),
      @ApiResponse(responseCode = "400", description = "Запрос не прошёл правила валидации (некорректный tbCode)",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "401", description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "403", description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "404", description = "Территориальный банк не найден",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))),
      @ApiResponse(responseCode = "500", description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)))
  })
  ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteById(
      @PathVariable("tbCode")
      @Parameter(
          name = "tbCode",
          in = ParameterIn.PATH,
          description = "Код территориального банка",
          required = true,
          schema = @Schema(
              title = "Код банка",
              description = "Код территориального банка",
              type = "string",
              pattern = "^.*$",
              maxLength = 255,
              example = "5500"
          )
      )
      String tbCode
  );
```
