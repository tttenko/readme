```java

/** Все единицы измерения */
    @NonNull
    public List<UomBankDto> getAllUoms() {
        return adapterCacheOps.getAllUoms();
    }

    /** Единицы измерения по списку кодов */
    @NonNull
    public List<UomBankDto> getUomByCodes(@NonNull List<String> uomCodes) {
        if (uomCodes.isEmpty()) {
            // на твой выбор: вернуть пустой список или кинуть BadRequest
            return List.of();
        }
        return cacheGetOrLoadService.fetchData(UOM_BY_CODE, uomCodes);
    }
```
