```java

@Test
void getAllCurrencies_thenLoadAllFromCache() {
    // given
    List<CurrencyDto> all = List.of(
            mock(CurrencyDto.class),
            mock(CurrencyDto.class)
    );
    when(currencyCacheOps.loadAllCurrencies()).thenReturn(all);

    // when
    List<CurrencyDto> result = currencyService2.getAllCurrencies();

    // then
    assertEquals(all, result);
    verify(currencyCacheOps).loadAllCurrencies();
    verifyNoInteractions(cacheGetOrLoadService);
}

@Test
void getCurrenciesByCodes_whenEmpty_thenReturnEmptyAndDoNotCallDependencies() {
    // given
    List<String> codes = List.of(); // пустой список

    // when
    List<CurrencyDto> result = currencyService2.getCurrenciesByCodes(codes);

    // then
    assertTrue(result.isEmpty());
    verifyNoInteractions(currencyCacheOps, cacheGetOrLoadService);
}

@Test
void getCurrenciesByCodes_whenNotEmpty_thenUseCacheGetOrLoadService() {
    // given
    List<String> codes = List.of("USD", "EUR");
    List<CurrencyDto> loaded = List.of(
            mock(CurrencyDto.class),
            mock(CurrencyDto.class)
    );

    when(cacheGetOrLoadService.fetchData(
            CurrencyService.CURRENCY_BY_CODE,
            codes
    )).thenReturn(loaded);

    // when
    List<CurrencyDto> result = currencyService2.getCurrenciesByCodes(codes);

    // then
    assertEquals(loaded, result);
    verify(cacheGetOrLoadService).fetchData(CurrencyService.CURRENCY_BY_CODE, codes);
    verifyNoInteractions(currencyCacheOps);
}

```
