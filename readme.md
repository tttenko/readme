```java

// все ТБ
@GetMapping("/tb/all")
@Operation(
        operationId = "getAllTerBanks",
        summary = "Получение списка всех Территориальных банков"
)
public ResultObj<List<TerBankDto>> getAllTerBanks() {
    return responseHandler.executeOrThrow(() ->
            getSuccessResponse(terBankService.getAllTerBanks()));
}

@GetMapping("/tb-requisite/all")
@Operation(
        operationId = "getAllTerBanksRequisite",
        summary = "Получение списка всех Территориальных банков с реквизитами"
)
public ResultObj<List<TerBankWithRequisiteDto>> getTerBanksRequisiteAll() {
    return responseHandler.executeOrThrow(() ->
            getSuccessResponse(terBankService.getAllTerBanksWithRequisite()));
}
```
