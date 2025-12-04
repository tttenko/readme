```java

@NonNull
    public List<TerBankDto> getAllTerBanks() {
        return terBankCache.getAllBanks();
    }

    /** Тербанки по списку кодов */
    @NonNull
    public List<TerBankDto> getTerBanksByCodes(@NonNull List<String> tbCodes) {
        if (tbCodes.isEmpty()) {
            return List.of(); // или кинуть BadRequestException — как у вас принято
        }
        return batchLoad.fetchData(TB_BY_CODE, tbCodes);
    }

 @NonNull
    public List<TerBankWithRequisiteDto> getAllTerBanksWithRequisite() {
        return terBankCache.getAllBanksWithRequisite();
    }

    /** Тербанки с реквизитами по списку кодов */
    @NonNull
    public List<TerBankWithRequisiteDto> getTerBanksWithRequisiteByCodes(@NonNull List<String> tbCodes) {
        if (tbCodes.isEmpty()) {
            return List.of();
        }
        return batchLoad.fetchData(TB_REQ_BY_CODE, tbCodes);
    }

```
