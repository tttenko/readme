```java

@Validated
@RequestMapping(value = "/api/v1/info/country", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
@Tag(
    name = "Country controller",
    description = "Сервис получения перечня стран из АС Мастер-данные"
)
@SecurityRequirement(name = "Authorization")
public interface UiCountryController {

  /**
   * Ищет страны по списку кодов ALPHA-2.
   */
  @GetMapping
  @Operation(
      operationId = "searchCountry",
      summary = "Получение перечня стран по массиву кодов ALPHA-2",
      description =
          "Возвращает список стран по заданным кодам ALPHA-2 (например: RU, US). "
              + "Параметр countryCode обязателен: должен содержать хотя бы одно значение; элементы списка не должны быть пустыми."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешный поиск стран",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректные параметры запроса (валидация, пустой/отсутствующий countryCode)",
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
  ResultObj<List<CountryDto>> searchCountryByCodes(
      @RequestParam(name = "countryCode")
      @NotEmpty(message = "Параметр countryCode должен содержать хотя бы одно значение")
      @ArraySchema(
          schema = @Schema(
              title = "Код страны (ALPHA-2)",
              description = "Код страны в формате ISO 3166-1 alpha-2",
              type = "string",
              minLength = 2,
              maxLength = 2,
              example = "RU"
          ),
          maxItems = 999
      )
      @Parameter(
          name = "countryCode",
          in = ParameterIn.QUERY,
          description =
              "Список кодов стран ALPHA-2 (ISO 3166-1). "
                  + "Можно передать несколько значений, повторяя query-параметр: "
                  + "`?countryCode=RU&countryCode=US`.",
          required = true
      )
      List<@NotBlank(message = "Код страны не должен быть пустым") String> countryCodes
  );

  /**
   * Ищет одну страну по коду ALPHA-2.
   */
  @GetMapping(value = "/{countryCode}")
  @Operation(
      operationId = "searchCountryByCode",
      summary = "Предоставление информации о стране по коду ALPHA-2",
      description =
          "Возвращает информацию о стране по коду ALPHA-2 (например: RU). "
              + "Параметр countryCode обязателен и не должен быть пустым."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение информации о стране",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректный код страны (валидация параметра)",
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
  ResultObj<List<CountryDto>> searchCountryByCode(
      @PathVariable("countryCode")
      @NotBlank(message = "countryCode не должен быть пустым")
      @Parameter(
          name = "countryCode",
          in = ParameterIn.PATH,
          description = "Код страны в формате ISO 3166-1 alpha-2 (например: RU)",
          required = true,
          schema = @Schema(type = "string", minLength = 2, maxLength = 2, example = "RU")
      )
      String countryCode
  );

  /**
   * Возвращает полный список всех стран.
   */
  @GetMapping(value = "/all")
  @Operation(
      operationId = "getAllCountries",
      summary = "Получение полного перечня стран",
      description = "Возвращает полный список всех стран из АС Мастер-данные."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение списка стран",
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
  ResultObj<List<CountryDto>> getAllCountries();
}
```
