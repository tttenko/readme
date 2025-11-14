```java

/**
 * Юнит-тесты для TerBankService2.
 * CacheGetOrLoadService и TerBankCacheOps замоканы.
 */
@ExtendWith(MockitoExtension.class)
class TerBankService2Test {

    @Mock
    CacheGetOrLoadService batchLoad;

    @Mock
    TerBankCacheOps terBankCache;

    TerBankService2 service;

    @BeforeEach
    void setUp() {
        service = new TerBankService2(batchLoad, terBankCache);
    }

    // ======================================================================
    // getTerBank
    // ======================================================================

    @Test
    void getTerBank_nullCodes_usesCacheAllBanks_andNotBatchLoad() {
        // given
        List<TerBankDto> allBanks = List.of(new TerBankDto(), new TerBankDto());
        when(terBankCache.getAllBanks()).thenReturn(allBanks);

        // when
        ResultObj<List<TerBankDto>> result = service.getTerBank(null);

        // then
        assertNotNull(result);
        // TODO: если у ResultObj другой геттер, замени getData() на свой метод
        assertEquals(allBanks, result.getData());

        verify(terBankCache).getAllBanks();
        verifyNoInteractions(batchLoad);
    }

    @Test
    void getTerBank_emptyCodes_usesCacheAllBanks_andNotBatchLoad() {
        // given
        List<String> tbCodes = Collections.emptyList();
        List<TerBankDto> allBanks = List.of(new TerBankDto());
        when(terBankCache.getAllBanks()).thenReturn(allBanks);

        // when
        ResultObj<List<TerBankDto>> result = service.getTerBank(tbCodes);

        // then
        assertNotNull(result);
        assertEquals(allBanks, result.getData());

        verify(terBankCache).getAllBanks();
        verifyNoInteractions(batchLoad);
    }

    @Test
    void getTerBank_nonEmptyCodes_usesBatchLoad_withTbByCode_andNotCacheAllBanks() {
        // given
        List<String> tbCodes = List.of("001", "002");
        List<TerBankDto> loaded = List.of(new TerBankDto());
        when(batchLoad.fetchData(TerBankService2.TB_BY_CODE, tbCodes)).thenReturn(loaded);

        // when
        ResultObj<List<TerBankDto>> result = service.getTerBank(tbCodes);

        // then
        assertNotNull(result);
        assertEquals(loaded, result.getData());

        verify(batchLoad).fetchData(TerBankService2.TB_BY_CODE, tbCodes);
        verify(terBankCache, never()).getAllBanks();
        verify(terBankCache, never()).getAllBanksWithRequisite();
    }

    // ======================================================================
    // getTerBankRequisite
    // ======================================================================

    @Test
    void getTerBankRequisite_nullCodes_usesCacheAllBanksWithRequisite_andNotBatchLoad() {
        // given
        List<TerBankWithRequisiteDto> all = List.of(new TerBankWithRequisiteDto());
        when(terBankCache.getAllBanksWithRequisite()).thenReturn(all);

        // when
        ResultObj<List<TerBankWithRequisiteDto>> result = service.getTerBankRequisite(null);

        // then
        assertNotNull(result);
        assertEquals(all, result.getData());

        verify(terBankCache).getAllBanksWithRequisite();
        verifyNoInteractions(batchLoad);
    }

    @Test
    void getTerBankRequisite_emptyCodes_usesCacheAllBanksWithRequisite_andNotBatchLoad() {
        // given
        List<String> tbCodes = Collections.emptyList();
        List<TerBankWithRequisiteDto> all = List.of(new TerBankWithRequisiteDto());
        when(terBankCache.getAllBanksWithRequisite()).thenReturn(all);

        // when
        ResultObj<List<TerBankWithRequisiteDto>> result = service.getTerBankRequisite(tbCodes);

        // then
        assertNotNull(result);
        assertEquals(all, result.getData());

        verify(terBankCache).getAllBanksWithRequisite();
        verifyNoInteractions(batchLoad);
    }

    @Test
    void getTerBankRequisite_nonEmptyCodes_usesBatchLoad_withTbReqByCode_andNotCache() {
        // given
        List<String> tbCodes = List.of("001", "002");
        List<TerBankWithRequisiteDto> loaded = List.of(new TerBankWithRequisiteDto());
        when(batchLoad.fetchData(TerBankService2.TB_REQ_BY_CODE, tbCodes)).thenReturn(loaded);

        // when
        ResultObj<List<TerBankWithRequisiteDto>> result = service.getTerBankRequisite(tbCodes);

        // then
        assertNotNull(result);
        assertEquals(loaded, result.getData());

        verify(batchLoad).fetchData(TerBankService2.TB_REQ_BY_CODE, tbCodes);
        verify(terBankCache, never()).getAllBanksWithRequisite();
        verify(terBankCache, never()).getAllBanks();
    }
}
```
