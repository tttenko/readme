```java

@Tag(
    name = "Common controller",
    description = "REST API для работы с сервисом АС Мастер-данные (UOM / Material / MaterialType)."
)
@SecurityRequirement(name = "Authorization")
@RequestMapping(value = "/api/v1/info", produces = MediaType.APPLICATION_JSON_VALUE)
public interface UiAdapterController {

  // -------------------- UOM --------------------

  /**
   * Получение всех доступных единиц измерения (UOM).
   */
  @GetMapping(value = "/uom/all")
  @Operation(
      operationId = "getAllMeasures",
      summary = "Получить список всех единиц измерения",
      description =
          "Возвращает полный список всех доступных единиц измерения (UOM) из системы мастер-данных. "
              + "Метод используется для заполнения справочников/выпадающих списков."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjUomBankDtoList.class))
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "500",
          description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      )
  })
  ResultObj<List<UomBankDto>> getAllMeasures();


  /**
   * Получение списка единиц измерения (UOM) по кодам.
   * Если коды не переданы — возвращаются все доступные UOM.
   */
  @GetMapping(value = "/uom")
  @Operation(
      operationId = "getMeasuresByCodes",
      summary = "Предоставление единиц измерения по списку кодов",
      description =
          "Возвращает список единиц измерения (UOM) по заданному списку кодов. "
              + "Параметр uomCode можно передавать несколько раз в query: "
              + "`?uomCode=796&uomCode=778`. "
              + "Если параметр uomCode не задан — возвращаются все доступные единицы измерения."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjUomBankDtoList.class))
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Запрос не прошёл правила валидации",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "500",
          description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      )
  })
  ResultObj<List<UomBankDto>> getMeasuresByCodes(
      @RequestParam(required = false, name = "uomCode")
      @ArraySchema(
          schema = @Schema(
              title = "Код единицы измерения",
              description = "Код UOM (единицы измерения)",
              type = "string",
              pattern = "^.*$",
              maxLength = 255,
              example = "796"
          ),
          maxItems = 999
      )
      @Parameter(
          name = "uomCode",
          in = ParameterIn.QUERY,
          description =
              "Список кодов единиц измерения (UOM). "
                  + "Можно передать несколько значений, повторяя query-параметр: `?uomCode=796&uomCode=778`. "
                  + "Если параметр не задан — возвращаются все доступные единицы измерения.",
          required = false
      )
      List<String> uomCode
  );


  /**
   * Получение информации об одной единице измерения (UOM) по её коду.
   */
  @GetMapping(value = "/uom/{uomCode}")
  @Operation(
      operationId = "getMeasuresById",
      summary = "Предоставление единицы измерения по коду",
      description =
          "Возвращает информацию об одной единице измерения (UOM) по её коду (uomCode). "
              + "Если по коду ничего не найдено — возвращается ошибка 404."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjUomBankDtoList.class))
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Запрос не прошёл правила валидации (некорректный uomCode)",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "404",
          description = "Единица измерения не найдена",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "500",
          description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      )
  })
  ResultObj<List<UomBankDto>> getMeasuresById(
      @PathVariable(name = "uomCode")
      @Parameter(
          name = "uomCode",
          in = ParameterIn.PATH,
          description = "Код единицы измерения (UOM)",
          required = true,
          schema = @Schema(
              title = "Код UOM",
              description = "Код единицы измерения (UOM)",
              type = "string",
              pattern = "^.*$",
              maxLength = 255,
              example = "796"
          )
      )
      String uomCode
  );


  // -------------------- Material --------------------

  /**
   * Получение информации об основной записи материала по списку кодов.
   * Если коды не переданы — возвращаются все доступные материалы.
   */
  @GetMapping(value = "/material")
  @Operation(
      operationId = "getMaterialByCodes",
      summary = "Получение информации об основной записи материала",
      description =
          "Возвращает информацию об основной записи материала по заданному списку кодов. "
              + "Параметр materialCode можно передавать несколько раз: `?materialCode=0001&materialCode=0002`. "
              + "Если коды не указаны — возвращаются все доступные материалы."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjMaterialDtoList.class))
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Запрос не прошёл правила валидации",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "500",
          description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      )
  })
  ResultObj<List<MaterialDto>> getMaterialByCodes(
      @RequestParam(required = false, name = "materialCode")
      @ArraySchema(
          schema = @Schema(
              title = "Код материала",
              description = "Код материала",
              type = "string",
              pattern = "^.*$",
              maxLength = 255,
              example = "000000000000000001"
          ),
          maxItems = 999
      )
      @Parameter(
          name = "materialCode",
          in = ParameterIn.QUERY,
          description =
              "Список кодов материалов. "
                  + "Можно передать несколько значений, повторяя query-параметр: "
                  + "`?materialCode=0001&materialCode=0002`. "
                  + "Если параметр не задан — возвращаются все доступные материалы.",
          required = false
      )
      List<String> materialCodes
  );


  /**
   * Получение информации об основной записи материала по коду.
   */
  @GetMapping(value = "/material/{materialCode}")
  @Operation(
      operationId = "getMaterialById",
      summary = "Получение информации об основной записи материала по коду",
      description =
          "Возвращает информацию об основной записи материала по его коду (materialCode). "
              + "Если по коду ничего не найдено — возвращается ошибка 404."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjMaterialDtoList.class))
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Запрос не прошёл правила валидации (некорректный materialCode)",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "404",
          description = "Материал не найден",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "500",
          description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      )
  })
  ResultObj<List<MaterialDto>> getMaterialById(
      @PathVariable(name = "materialCode")
      @Parameter(
          name = "materialCode",
          in = ParameterIn.PATH,
          description = "Код материала",
          required = true,
          schema = @Schema(
              title = "Код материала",
              description = "Код материала",
              type = "string",
              pattern = "^.*$",
              maxLength = 255,
              example = "000000000000000001"
          )
      )
      String materialCode
  );


  // -------------------- MaterialType --------------------

  /**
   * Возвращает полный список всех типов материалов.
   */
  @GetMapping(value = "/material-type/all")
  @Operation(
      operationId = "getAllMaterialTypes",
      summary = "Получение информации о всех доступных видах ТМЦ",
      description =
          "Возвращает полный список всех доступных видов ТМЦ (типов материалов) из системы мастер-данных."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjMaterialTypeDtoList.class))
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "500",
          description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      )
  })
  ResultObj<List<MaterialTypeDto>> getAllMaterialTypes();


  /**
   * Возвращает список типов материалов по идентификаторам.
   * Если typeId не передан — возвращаются все доступные типы материалов.
   */
  @GetMapping(value = "/material-type")
  @Operation(
      operationId = "getMaterialTypeIds",
      summary = "Получение информации о виде ТМЦ по списку идентификаторов",
      description =
          "Возвращает список видов ТМЦ (типов материалов) по заданному списку идентификаторов. "
              + "Параметр typeId можно передавать несколько раз: `?typeId=1&typeId=2`. "
              + "Если параметр typeId не задан — возвращаются все доступные виды ТМЦ."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjMaterialTypeDtoList.class))
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Запрос не прошёл правила валидации",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "500",
          description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      )
  })
  ResultObj<List<MaterialTypeDto>> getMaterialTypeIds(
      @RequestParam(name = "typeId", required = false)
      @ArraySchema(
          schema = @Schema(
              title = "Идентификатор вида ТМЦ",
              description = "Идентификатор вида ТМЦ (тип материала)",
              type = "string",
              pattern = "^.*$",
              maxLength = 255,
              example = "1"
          ),
          maxItems = 999
      )
      @Parameter(
          name = "typeId",
          in = ParameterIn.QUERY,
          description =
              "Список идентификаторов видов ТМЦ. "
                  + "Можно передать несколько значений, повторяя query-параметр: `?typeId=1&typeId=2`. "
                  + "Если параметр не задан — возвращаются все доступные виды ТМЦ.",
          required = false
      )
      List<String> typeIds
  );


  /**
   * Возвращает информацию о виде ТМЦ по его идентификатору.
   */
  @GetMapping(value = "/material-type/{typeId}")
  @Operation(
      operationId = "getMaterialTypeById",
      summary = "Получение информации о виде ТМЦ по идентификатору",
      description =
          "Возвращает информацию о виде ТМЦ (типе материала) по его идентификатору (typeId). "
              + "Если по идентификатору ничего не найдено — возвращается ошибка 404."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешно",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjMaterialTypeDtoList.class))
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Запрос не прошёл правила валидации (некорректный typeId)",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "401",
          description = "Пользователь не аутентифицирован",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "403",
          description = "Нет прав на операцию",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "404",
          description = "Вид ТМЦ не найден",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      ),
      @ApiResponse(
          responseCode = "500",
          description = "На сервере произошла ошибка",
          content = @Content(mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class))
      )
  })
  ResultObj<List<MaterialTypeDto>> getMaterialTypeById(
      @PathVariable(name = "typeId")
      @Parameter(
          name = "typeId",
          in = ParameterIn.PATH,
          description = "Идентификатор вида ТМЦ (тип материала)",
          required = true,
          schema = @Schema(
              title = "Идентификатор вида ТМЦ",
              description = "Идентификатор вида ТМЦ (тип материала)",
              type = "string",
              pattern = "^.*$",
              maxLength = 255,
              example = "1"
          )
      )
      String typeId
  );


  // --------- helper schemas to show generics in OpenAPI ---------
  @Schema(name = "ResultObjUomBankDtoList")
  class ResultObjUomBankDtoList extends ResultObj<List<UomBankDto>> {}

  @Schema(name = "ResultObjMaterialDtoList")
  class ResultObjMaterialDtoList extends ResultObj<List<MaterialDto>> {}

  @Schema(name = "ResultObjMaterialTypeDtoList")
  class ResultObjMaterialTypeDtoList extends ResultObj<List<MaterialTypeDto>> {}
}

```
