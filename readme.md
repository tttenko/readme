```java

@RequestMapping(value = "/api/v1/info", produces = MediaType.APPLICATION_JSON_VALUE)
@Tag(name = "Common controller",
        description = "REST API для работы с сервисом АС Мастер-данные")
public interface AdapterController {

    /**
     * Documenting translate JavaDoc Documenting rebuild JavaDoc
     * Получение всех доступных единиц измерения.
     *
     * @return Результат в виде объекта ResultObj.
     */
    @GetMapping(value = "/uom/all")
    @Operation(
            operationId = "getAllMeasures",
            summary = "Получить список всех единиц измерения."
    )
    ResultObj<List<UomBankDto>> getAllMeasures();

    /**
     * Documenting translate JavaDoc Documenting rebuild JavaDoc
     * Получение списка единиц измерения (UOM) по кодам.
     *
     * @param uomCode Список кодов единиц измерения (необязательный параметр)
     * @return Результат в виде объекта ResultObj
     */
    @GetMapping(value = "/uom")
    @Operation(operationId = "getMeasuresByCodes",
            summary = "Предоставления единиц измерения по списку кодов.")
    ResultObj<List<UomBankDto>> getMeasuresByCodes(
            @RequestParam(name = "uomCode")
            @NotEmpty(message = "Параметр uomCode должен содержать хотя бы одно значение")
            List<@NotBlank(message = "Код единицы измерения не должен быть пустым") String> uomCode);

    /**
     * Documenting translate JavaDoc Documenting rebuild JavaDoc
     * Получение информации об одной единице измерения (UOM) по её коду.
     *
     * @param uomCode Код единицы измерения
     * @return Результат в виде объекта ResultObj
     */
    @GetMapping(value = "/uom/{uomCode}")
    @Operation(operationId = "getMeasuresById",
            summary = "Предоставление единицы измерения по коду.")
    ResultObj<List<UomBankDto>> getMeasuresById(@PathVariable(name = "uomCode") String uomCode);

    /**
     * Documenting translate JavaDoc Documenting rebuild JavaDoc
     * Получение информации о материале по его коду.
     *
     * @param materialCodes Код материала
     * @return Результат в виде объекта ResultObj
     */
    @GetMapping(value = "/material")
    @Operation(operationId = "getMaterialByCodes",
            summary = "Получение информации об основной записи материала.")
    ResultObj<List<MaterialDto>> getMaterialByCodes(
            @RequestParam(name = "materialCode")
            @NotEmpty(message = "Параметр materialCode должен содержать хотя бы одно значение")
            List<@NotBlank(message = "Код материала не должен быть пустым") String> materialCodes);

    /**
     * Documenting translate JavaDoc Documenting rebuild JavaDoc
     * Получение информации о материалах по их кодам.
     * Если коды не указаны, возвращает все доступные материалы.
     *
     * @param materialCode Список кодов материалов (необязательный параметр)
     * @return Результат в виде объекта ResultObj
     */
    @GetMapping(value = "/material/{materialCode}")
    @Operation(operationId = "getMaterialById",
            summary = "Получение информации об основной записи материала.")
    ResultObj<List<MaterialDto>> getMaterialById(@PathVariable(name = "materialCode") String materialCode);

    /**
     * Documenting translate JavaDoc Documenting rebuild JavaDoc
     * Возвращает полный список всех материалов из системы мастер-данных.
     *
     * @return объект результата с полным списком материалов
     */
    @GetMapping(value = "/material/all")
    @Operation(operationId = "getAllMaterials",
            summary = "Получение полного списка всех материалов.")
    ResultObj<List<MaterialDto>> getAllMaterials();

    /**
     * Documenting translate JavaDoc Documenting rebuild JavaDoc
     * Возвращает полный список всех типов материалов из системы мастер-данных.
     *
     * @return объект результата с полным списком типов материалов
     */
    @GetMapping(value = "/material-type/all")
    @Operation(
            operationId = "getAllMaterialTypes",
            summary = "Получение информации о всех доступных видах ТМЦ")
    ResultObj<List<MaterialTypeDto>> getAllMaterialTypes();

    /**
     * Documenting translate JavaDoc Documenting rebuild JavaDoc
     * Получение информации о видах ТМЦ по их идентификаторам.
     *
     * @param typeId Список идентификаторов видов ТМЦ (необязательный параметр)
     * @return Результат в виде объекта ResultObj
     */
    @GetMapping(value = "/material-type")
    @Operation(operationId = "getMaterialTypeIds",
            summary = "Получение информации о виде ТМЦ по списку идентификаторов")
    ResultObj<List<MaterialTypeDto>> getMaterialTypeIds(
            @RequestParam(name = "typeId")
            @NotEmpty(message = "Параметр typeId должен содержать хотя бы одно значение")
            List<@NotBlank(message = "Идентификатор вида ТМЦ не должен быть пустым") String> typeIds);

    /**
     * Documenting translate JavaDoc Documenting rebuild JavaDoc
     * Получение информации о виде ТМЦ по его идентификатору.
     *
     * @param typeId Идентификатор вида ТМЦ
     * @return Результат в виде объекта ResultObj
     */
    @GetMapping(value = "/material-type/{typeId}")
    @Operation(operationId = "getMaterialTypeById",
            summary = "Получение информации о виде ТМЦ по идентификатору")
    ResultObj<List<MaterialTypeDto>> getMaterialTypeById(@PathVariable(name = "typeId") String typeId);
}

```
