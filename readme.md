```java

// все материалы
@NonNull
public List<MaterialDto> getAllMaterials() {
    return adapterCacheOps.getAllMaterials();
}

// материалы по списку кодов
@NonNull
public List<MaterialDto> getMaterialByCodes(@NonNull List<String> materialCodes) {
    if (materialCodes.isEmpty()) {
        // на выбор: вернуть пустой список или бросить исключение
        return List.of();
    }
    return cacheGetOrLoadService.fetchData(MATERIAL_BY_CODE, materialCodes);
}
```
