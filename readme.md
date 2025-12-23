```java

@Validated
@RequestMapping(value = "/api/v1/info", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
@Tag(
    name = "Region controller",
    description = "Сервис получения перечня регионов из АС Мастер-данные"
)
@SecurityRequirement(name = "Authorization")
public interface UiRegionController {

  /**
   * Ищет регионы по переданному списку кодов.
   */
  @GetMapping(value = "/region_code")
  @Operation(
      operationId = "searchRegion",
      summary = "Получение перечня регионов по массиву кодов регионов",
      description =
          "Возвращает список регионов по заданным кодам (например: 77). "
              + "Параметр regionCode обязателен: должен содержать хотя бы одно значение; "
              + "элементы списка не должны быть пустыми."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешный поиск регионов",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректные параметры запроса (валидация, пустой/отсутствующий regionCode)",
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
  ResultObj<List<RegionDto>> searchRegionsByCode(
      @RequestParam(name = "regionCode")
      @NotEmpty(message = "Параметр regionCode должен содержать хотя бы одно значение")
      @ArraySchema(
          schema = @Schema(
              title = "Код региона",
              description = "Код региона (например: 77)",
              type = "string",
              pattern = "^\\d{1,10}$",
              maxLength = 10,
              example = "77"
          ),
          maxItems = 999
      )
      @Parameter(
          name = "regionCode",
          in = ParameterIn.QUERY,
          description =
              "Список кодов регионов. "
                  + "Можно передать несколько значений, повторяя query-параметр: "
                  + "`?regionCode=77&regionCode=78`. "
                  + "Параметр обязателен и не должен быть пустым.",
          required = true
      )
      List<@NotBlank(message = "Код региона не должен быть пустым") String> regionCodes
  );

  /**
   * Возвращает информацию о регионе по его коду.
   */
  @GetMapping(value = "/region_code/{regionCode}")
  @Operation(
      operationId = "searchRegionByCode",
      summary = "Предоставление информации о регионе по коду региона",
      description =
          "Возвращает информацию о регионе по его коду (например: 77). "
              + "Параметр regionCode обязателен и не должен быть пустым."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение информации о регионе",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObj.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректный код региона (валидация параметра)",
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
  ResultObj<List<RegionDto>> searchRegionByCode(
      @PathVariable("regionCode")
      @NotBlank(message = "regionCode не должен быть пустым")
      @Parameter(
          name = "regionCode",
          in = ParameterIn.PATH,
          description = "Код региона (например: 77)",
          required = true,
          schema = @Schema(
              title = "Код региона",
              description = "Код региона (например: 77)",
              type = "string",
              pattern = "^\\d{1,10}$",
              maxLength = 10,
              example = "77"
          )
      )
      String regionCode
  );

  /**
   * Возвращает полный список всех регионов.
   */
  @GetMapping(value = "/region_code/all")
  @Operation(
      operationId = "getAllRegions",
      summary = "Получение полного перечня регионов",
      description =
          "Возвращает полный список всех регионов из системы мастер-данных. "
              + "Метод доступен по GET запросу на эндпоинт `/api/v1/info/region_code/all`. "
              + "Поддерживает кеширование на уровне сервиса для повышения производительности."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешное получение списка регионов",
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
  ResultObj<List<RegionDto>> getAllRegions();
}
```
