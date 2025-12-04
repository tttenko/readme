```java

@GetMapping(value = "/uom/all", produces = APPLICATION_JSON_UTF8_VALUE)
@Operation(
        operationId = "getAllMeasures",
        summary = "Получить список всех единиц измерения"
)
public ResultObj<List<UomBankDto>> getAllMeasures() {
    return getSuccessResponse(adapterService.getAllUoms());
}

```
