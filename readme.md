```java

@ExtendWith(MockitoExtension.class)
class CurrencyService2Test {

    @Mock
    private CurrencyCacheOps currencyCacheOps;

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    @InjectMocks
    private CurrencyService2 currencyService2;

    @Test
    void givenNullCodes_whenSearchCurrenciesByCode_thenLoadAllFromCache() {
        // given
        List<CurrencyDto> all = List.of(
                mock(CurrencyDto.class),
                mock(CurrencyDto.class)
        );
        when(currencyCacheOps.loadAllCurrencies()).thenReturn(all);

        // when
        ResultObj<List<CurrencyDto>> result =
                currencyService2.searchCurrenciesByCode(null);

        // then
        assertEquals(all, result.getData());

        verify(currencyCacheOps).loadAllCurrencies();
        verifyNoInteractions(cacheGetOrLoadService);
    }

    @Test
    void givenEmptyCodes_whenSearchCurrenciesByCode_thenLoadAllFromCache() {
        // given
        List<CurrencyDto> all = List.of(mock(CurrencyDto.class));
        when(currencyCacheOps.loadAllCurrencies()).thenReturn(all);

        // when
        ResultObj<List<CurrencyDto>> result =
                currencyService2.searchCurrenciesByCode(List.of());

        // then
        assertEquals(all, result.getData());

        verify(currencyCacheOps).loadAllCurrencies();
        verifyNoInteractions(cacheGetOrLoadService);
    }

    @Test
    void givenCodes_whenSearchCurrenciesByCode_thenUseCacheGetOrLoadService() {
        // given
        List<String> codes = List.of("USD", "EUR");
        List<CurrencyDto> loaded = List.of(
                mock(CurrencyDto.class),
                mock(CurrencyDto.class)
        );

        when(cacheGetOrLoadService.fetchData(
                CurrencyService2.CURRENCY_BY_CODE,
                codes
        )).thenReturn(loaded);

        // when
        ResultObj<List<CurrencyDto>> result =
                currencyService2.searchCurrenciesByCode(codes);

        // then
        assertEquals(loaded, result.getData());

        verify(cacheGetOrLoadService)
                .fetchData(CurrencyService2.CURRENCY_BY_CODE, codes);
        verifyNoInteractions(currencyCacheOps);
    }
}
```
