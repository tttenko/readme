```java

/**
   * Получение действующих ставок НДС на заданную дату.
   * Если дата не передана — используется текущая дата/время сервера.
   * Параметр rate (если задан) фильтрует результат по ставкам.
   */
  @GetMapping("/main-nds")
  @Operation(
      operationId = "getNdsByRate",
      summary = "Получение действующих ставок НДС на дату",
      description =
          "Возвращает список действующих ставок НДС на указанную дату/время. "
              + "Если параметр date не задан — используется текущее время сервера. "
              + "Если параметр rate задан — результат фильтруется по указанным ставкам."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешно",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjNdsDtoList.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Запрос не прошёл правила валидации (некорректные параметры)",
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
          description = "На сервере произошла ошибка",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      )
  })
  ResultObj<List<NdsDto>> getNdsByRate(
      @Parameter(
          name = "rate",
          in = ParameterIn.QUERY,
          description =
              "Фильтр по ставкам НДС. Можно передать несколько значений параметром, повторяя query-параметр. "
                  + "Например: ?rate=20&rate=10",
          required = false,
          array = @ArraySchema(schema = @Schema(type = "string")),
          examples = {
              @io.swagger.v3.oas.annotations.media.ExampleObject(name = "Single", value = "20"),
              @io.swagger.v3.oas.annotations.media.ExampleObject(name = "Multi", value = "[\"20\",\"10\"]")
          }
      )
      @RequestParam(name = "rate", required = false)
      List<String> rate,

      @Parameter(
          name = "date",
          in = ParameterIn.QUERY,
          description =
              "Дата/время, на которую нужно определить действующие ставки. "
                  + "Формат ISO-8601 (ZonedDateTime). "
                  + "Если не передан — используется текущее время сервера.",
          required = false,
          schema = @Schema(type = "string", format = "date-time"),
          example = "2025-07-21T10:00:00+03:00"
      )
      @RequestParam(name = "date", required = false)
      @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
      ZonedDateTime date
  );

  /**
   * Получение действующих основных ставок НДС по коду налога и/или ставке на заданную дату.
   * Если дата не передана — используется текущая дата/время сервера.
   */
  @GetMapping("/main-nds-code")
  @Operation(
      operationId = "getNdsByCode",
      summary = "Получение основных ставок НДС по коду налога на дату",
      description =
          "Возвращает список действующих основных ставок НДС с возможной фильтрацией по коду налога (code) "
              + "и/или ставке (rate) на указанную дату/время. "
              + "Если date не задан — используется текущее время сервера."
  )
  @ApiResponses({
      @ApiResponse(
          responseCode = "200",
          description = "Успешно",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = ResultObjNdsDtoList.class)
          )
      ),
      @ApiResponse(
          responseCode = "400",
          description = "Запрос не прошёл правила валидации (некорректные параметры)",
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
          description = "На сервере произошла ошибка",
          content = @Content(
              mediaType = MediaType.APPLICATION_JSON_VALUE,
              schema = @Schema(implementation = MessageObj.class)
          )
      )
  })
  ResultObj<List<NdsDto>> getNdsByCode(
      @Parameter(
          name = "code",
          in = ParameterIn.QUERY,
          description =
              "Код налога. Можно передать несколько значений, повторяя query-параметр. "
                  + "Например: ?code=01&code=02",
          required = false,
          array = @ArraySchema(schema = @Schema(type = "string")),
          examples = {
              @io.swagger.v3.oas.annotations.media.ExampleObject(name = "Single", value = "01"),
              @io.swagger.v3.oas.annotations.media.ExampleObject(name = "Multi", value = "[\"01\",\"02\"]")
          }
      )
      @RequestParam(name = "code", required = false)
      List<String> code,

      @Parameter(
          name = "rate",
          in = ParameterIn.QUERY,
          description = "Фильтр по ставкам НДС (см. метод /main-nds).",
          required = false,
          array = @ArraySchema(schema = @Schema(type = "string")),
          examples = {
              @io.swagger.v3.oas.annotations.media.ExampleObject(name = "Single", value = "20")
          }
      )
      @RequestParam(name = "rate", required = false)
      List<String> rate,

      @Parameter(
          name = "date",
          in = ParameterIn.QUERY,
          description = "Дата/время (ISO-8601, ZonedDateTime). Если не передан — используется текущее время сервера.",
          required = false,
          schema = @Schema(type = "string", format = "date-time"),
          example = "2025-07-21T10:00:00+03:00"
      )
      @RequestParam(name = "date", required = false)
      @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
      ZonedDateTime date
  );
```
