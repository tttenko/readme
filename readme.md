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

@Test
void getAllMaterialTypes_thenUsesAllFromCacheOps() {
    // given
    List<MaterialTypeDto> all = List.of(
            MaterialTypeDto.builder().typeId("TYPE1").build()
    );
    when(adapterCacheOps.getAllMaterialTypes()).thenReturn(all);

    // when
    List<MaterialTypeDto> result = adapterService2.getAllMaterialTypes();

    // then
    assertThat(result).containsExactlyElementsOf(all);
    verify(adapterCacheOps).getAllMaterialTypes();
    verifyNoInteractions(cacheGetOrLoadService);
}

@Test
void getMaterialTypeByIds_whenIdsEmpty_thenReturnsEmptyListAndDoesNotCallDependencies() {
    // given
    List<String> ids = List.of();

    // when
    List<MaterialTypeDto> result = adapterService2.getMaterialTypeByIds(ids);

    // then
    assertThat(result).isEmpty();
    verifyNoInteractions(adapterCacheOps, cacheGetOrLoadService);
}

@SuppressWarnings("unchecked")
@Test
void getMaterialTypeByIds_whenIdsNotEmpty_thenUsesCacheGetOrLoadService() {
    // given
    List<String> ids = List.of("MT1", "MT2");
    List<MaterialTypeDto> loaded = List.of(
            MaterialTypeDto.builder().typeId("MT1").build(),
            MaterialTypeDto.builder().typeId("MT2").build()
    );

    when(cacheGetOrLoadService.fetchData(
            eq(AdapterService.MATERIAL_TYPE_BY_ID),
            eq(ids)
    )).thenReturn((List) loaded);

    // when
    List<MaterialTypeDto> result = adapterService2.getMaterialTypeByIds(ids);

    // then
    assertThat(result).containsExactlyElementsOf(loaded);
    verify(cacheGetOrLoadService).fetchData(AdapterService.MATERIAL_TYPE_BY_ID, ids);
    verifyNoInteractions(adapterCacheOps);
}

@Test
void getAllMaterials_thenUsesAllFromCacheOps() {
    // given
    List<MaterialDto> all = List.of(
            MaterialDto.builder().materialCode("M1").build()
    );
    when(adapterCacheOps.getAllMaterials()).thenReturn(all);

    // when
    List<MaterialDto> result = adapterService2.getAllMaterials();

    // then
    assertThat(result).containsExactlyElementsOf(all);
    verify(adapterCacheOps).getAllMaterials();
    verifyNoInteractions(cacheGetOrLoadService);
}

@Test
void getMaterialByCodes_whenCodesEmpty_thenReturnsEmptyListAndDoesNotCallDependencies() {
    // given
    List<String> codes = List.of();

    // when
    List<MaterialDto> result = adapterService2.getMaterialByCodes(codes);

    // then
    assertThat(result).isEmpty();
    verifyNoInteractions(adapterCacheOps, cacheGetOrLoadService);
}

@SuppressWarnings("unchecked")
@Test
void getMaterialByCodes_whenCodesNotEmpty_thenUsesCacheGetOrLoadService() {
    // given
    List<String> codes = List.of("M1", "M2");
    List<MaterialDto> loaded = List.of(
            MaterialDto.builder().materialCode("M1").build(),
            MaterialDto.builder().materialCode("M2").build()
    );

    when(cacheGetOrLoadService.fetchData(
            eq(AdapterService.MATERIAL_BY_CODE),
            eq(codes)
    )).thenReturn((List) loaded);

    // when
    List<MaterialDto> result = adapterService2.getMaterialByCodes(codes);

    // then
    assertThat(result).containsExactlyElementsOf(loaded);
    verify(cacheGetOrLoadService).fetchData(AdapterService.MATERIAL_BY_CODE, codes);
    verifyNoInteractions(adapterCacheOps);
}
```
