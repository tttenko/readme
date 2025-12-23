```java

@Tag(
    name = "Currency controller",
    description = "Сервис получения валют из АС Мастер-данные"
)
@SecurityRequirement(name = "Authorization")
@RequestMapping(value = "/api/v1/info/currency", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
public interface UiCurrencyController {

  /**
   * Возвращает полный список всех валют из системы мастер-данных.
   * Поддерживает автоматическое кеширование на уровне сервиса.
   */
  @GetMapping(value = "/all")
  @Operation(
      operationId = "getAllCurrencies",
      summary = "Предоставление информации обо всех доступных валютах",
      description =
          "Возвращает полный список всех валют из системы мастер-данных. "
              + "Поддерживает автоматическое кеширование на уровне сервиса."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение полного списка валют",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
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
  ResultObj<List<CurrencyDto>> getAllCurrencies();


  /**
   * Возвращает валюты по заданным кодам (например, "RUB", "USD").
   * Поддерживает массовый поиск. Если запрашиваемый код не найден — валюта не включается в результат.
   */
  @GetMapping
  @Operation(
      operationId = "searchCurrenciesByCode",
      summary = "Предоставление информации о валютах по списку кодов",
      description =
          "Возвращает список валют по заданным кодам (например: RUB, USD). "
              + "Поддерживает массовый поиск: параметр currencyCode можно повторять в query, "
              + "например: `?currencyCode=RUB&currencyCode=USD`. "
              + "Если валюта по какому-либо коду не найдена — она не включается в результат. "
              + "Если параметр currencyCode не задан — возвращается список всех валют."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение списка валют",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректные параметры запроса (валидация)",
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
  ResultObj<List<CurrencyDto>> searchCurrenciesByCodes(
      @RequestParam(required = false, name = "currencyCode")
      @ArraySchema(
          schema = @Schema(
              title = "Код валюты",
              description = "Код валюты (например: RUB, USD)",
              type = "string",
              pattern = "^[A-Z]{3}$",
              minLength = 3,
              maxLength = 3,
              example = "RUB"
          ),
          maxItems = 999
      )
      @Parameter(
          name = "currencyCode",
          in = ParameterIn.QUERY,
          description =
              "Список кодов валют. "
                  + "Можно передать несколько значений, повторяя query-параметр: "
                  + "`?currencyCode=RUB&currencyCode=USD`. "
                  + "Если параметр не задан — возвращаются все валюты.",
          required = false
      )
      List<String> currencyCodes
  );


  /**
   * Получает информацию о валюте по её коду.
   * Если валюта не найдена — выбрасывается MdaDataNotFoundException (404).
   */
  @GetMapping(value = "/{currencyCode}")
  @Operation(
      operationId = "getCurrenciesByCodes",
      summary = "Предоставление информации о валюте по коду",
      description =
          "Возвращает информацию о валюте по её коду (например: RUB). "
              + "Если валюта не найдена — возвращается ошибка 404."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение информации о валюте",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректный код валюты (валидация параметра)",
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
          description = "Валюта не найдена",
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
  ResultObj<List<CurrencyDto>> searchCurrencyByCode(
      @PathVariable("currencyCode")
      @Parameter(
          name = "currencyCode",
          in = ParameterIn.PATH,
          description = "Код валюты (например: RUB)",
          required = true,
          schema = @Schema(
              title = "Код валюты",
              description = "Код валюты (например: RUB, USD)",
              type = "string",
              pattern = "^[A-Z]{3}$",
              minLength = 3,
              maxLength = 3,
              example = "RUB"
          )
      )
      String currencyCode
  );
}

```
