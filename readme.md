```java

@NonNull
public List<MaterialTypeDto> getAllMaterialTypes() {
    return adapterCacheOps.getAllMaterialTypes();
}

@NonNull
public List<MaterialTypeDto> getMaterialTypeByIds(@NonNull List<String> typeIds) {
    if (typeIds.isEmpty()) {
        // либо бросаешь BadRequest, либо возвращаешь пустой список
        return List.of();
    }
    return cacheGetOrLoadService.fetchData(MATERIAL_TYPE_BY_ID, typeIds);
}

@GetMapping("/material-type/all")
@Operation(
        operationId = "getAllMaterialTypes",
        summary = "Получение информации о всех доступных видах ТМЦ"
)
public ResultObj<List<MaterialTypeDto>> getAllMaterialTypes() {
    return getSuccessResponse(adapterService.getAllMaterialTypes());
}
```
