```java

@Test
void getAllTerBanks_usesCacheAllBanks_andNotBatchLoad() {
    // given
    List<TerBankDto> allBanks = List.of(new TerBankDto(), new TerBankDto());
    when(terBankCache.getAllBanks()).thenReturn(allBanks);

    // when
    List<TerBankDto> result = service.getAllTerBanks();

    // then
    assertNotNull(result);
    assertEquals(allBanks, result);
    verify(terBankCache).getAllBanks();
    verifyNoInteractions(batchLoad);
}

@Test
void getTerBanksByCodes_emptyCodes_returnsEmpty_andNotCallDependencies() {
    // given
    List<String> tbCodes = Collections.emptyList();

    // when
    List<TerBankDto> result = service.getTerBanksByCodes(tbCodes);

    // then
    assertNotNull(result);
    assertTrue(result.isEmpty());
    verifyNoInteractions(terBankCache, batchLoad);
}

@Test
void getTerBanksByCodes_nonEmptyCodes_usesBatchLoad_withTbByCode_andNotCacheAllBanks() {
    // given
    List<String> tbCodes = List.of("001", "002");
    List<TerBankDto> loaded = List.of(new TerBankDto(), new TerBankDto());
    when(batchLoad.fetchData(TerBankService.TB_BY_CODE, tbCodes)).thenReturn(loaded);

    // when
    List<TerBankDto> result = service.getTerBanksByCodes(tbCodes);

    // then
    assertNotNull(result);
    assertEquals(loaded, result);
    verify(batchLoad).fetchData(TerBankService.TB_BY_CODE, tbCodes);
    verify(terBankCache, never()).getAllBanks();
    verify(terBankCache, never()).getAllBanksWithRequisite();
}

// ================== Тербанки с реквизитами ==================

@Test
void getAllTerBanksWithRequisite_usesCacheAllBanksWithRequisite_andNotBatchLoad() {
    // given
    List<TerBankWithRequisiteDto> all = List.of(
            new TerBankWithRequisiteDto(),
            new TerBankWithRequisiteDto()
    );
    when(terBankCache.getAllBanksWithRequisite()).thenReturn(all);

    // when
    List<TerBankWithRequisiteDto> result = service.getAllTerBanksWithRequisite();

    // then
    assertNotNull(result);
    assertEquals(all, result);
    verify(terBankCache).getAllBanksWithRequisite();
    verifyNoInteractions(batchLoad);
}

@Test
void getTerBanksWithRequisiteByCodes_emptyCodes_returnsEmpty_andNotCallDependencies() {
    // given
    List<String> tbCodes = Collections.emptyList();

    // when
    List<TerBankWithRequisiteDto> result = service.getTerBanksWithRequisiteByCodes(tbCodes);

    // then
    assertNotNull(result);
    assertTrue(result.isEmpty());
    verifyNoInteractions(terBankCache, batchLoad);
}

@Test
void getTerBanksWithRequisiteByCodes_nonEmptyCodes_usesBatchLoad_withTbReqByCode_andNotCacheAll() {
    // given
    List<String> tbCodes = List.of("001", "002");
    List<TerBankWithRequisiteDto> loaded = List.of(
            new TerBankWithRequisiteDto(),
            new TerBankWithRequisiteDto()
    );
    when(batchLoad.fetchData(TerBankService.TB_REQ_BY_CODE, tbCodes)).thenReturn(loaded);

    // when
    List<TerBankWithRequisiteDto> result = service.getTerBanksWithRequisiteByCodes(tbCodes);

    // then
    assertNotNull(result);
    assertEquals(loaded, result);
    verify(batchLoad).fetchData(TerBankService.TB_REQ_BY_CODE, tbCodes);
    verify(terBankCache, never()).getAllBanksWithRequisite();
    verify(terBankCache, never()).getAllBanks();
}

```
