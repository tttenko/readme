```java

@ExtendWith(MockitoExtension.class)
class LoaderCurrencyByCodeTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SearchRequestProperties properties;

    @Mock
    private CurrencyMapper currencyMapper;

    @InjectMocks
    private LoaderCurrencyByCode loader;

    // --- простые "конфигурационные" методы ---

    @Test
    void givenLoader_whenCacheName_thenReturnCurrencyByCode() {
        assertEquals(CurrencyService2.CURRENCY_BY_CODE, loader.cacheName());
    }

    @Test
    void givenLoader_whenElementType_thenReturnCurrencyDtoClass() {
        assertEquals(CurrencyDto.class, loader.elementType());
    }

    @Test
    void givenCurrencyDto_whenExtractKey_thenReturnCurrencyCode() {
        CurrencyDto dto = new CurrencyDto();
        dto.setCurrencyCode("USD");

        String key = loader.extractKey(dto);

        assertEquals("USD", key);
    }

    // --- основная логика fetchByKeys ---

    @Test
    void givenEmptyKeys_whenFetchByKeys_thenReturnEmptyAndSkipBackend() {
        // given
        List<String> keys = List.of();

        // when
        List<CurrencyDto> result = loader.fetchByKeys(keys);

        // then
        assertTrue(result.isEmpty());
        verifyNoInteractions(baseMasterDataRequestService, properties, currencyMapper);
    }

    @Test
    void givenCodes_whenFetchByKeys_thenDelegateToBackendAndMapResult() {
        // given
        List<String> codes = List.of("USD", "EUR");
        String slug = "currency";
        String attrId = "currencyCode";

        when(properties.getSlugValueForCurrency()).thenReturn(slug);
        when(properties.getCurrencyAttributeId()).thenReturn(attrId);

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestDataByAttributes(slug, attrId, codes))
                .thenReturn(resp);

        List<CurrencyDto> mapped = List.of(
                CurrencyDto.builder().currencyCode("USD").build(),
                CurrencyDto.builder().currencyCode("EUR").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper)
            ).thenReturn(mapped);

            // when
            List<CurrencyDto> result = loader.fetchByKeys(codes);

            // then
            assertEquals(mapped, result);

            verify(properties).getSlugValueForCurrency();
            verify(properties).getCurrencyAttributeId();
            verify(baseMasterDataRequestService, times(1))
                    .requestDataByAttributes(slug, attrId, codes);

            statics.verify(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper),
                    times(1));
        }
    }

    @Test
    void givenTwoSequentialCalls_whenFetchByKeys_thenBackendInvokedTwice_noCaching() {
        // given
        List<String> codes = List.of("JPY");
        String slug = "currency";
        String attrId = "currencyCode";

        when(properties.getSlugValueForCurrency()).thenReturn(slug);
        when(properties.getCurrencyAttributeId()).thenReturn(attrId);

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestDataByAttributes(slug, attrId, codes))
                .thenReturn(resp);

        List<CurrencyDto> mapped =
                List.of(CurrencyDto.builder().currencyCode("JPY").build());

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper)
            ).thenReturn(mapped);

            // when
            List<CurrencyDto> r1 = loader.fetchByKeys(codes);
            List<CurrencyDto> r2 = loader.fetchByKeys(codes);

            // then
            assertEquals(mapped, r1);
            assertEquals(mapped, r2);

            verify(baseMasterDataRequestService, times(2))
                    .requestDataByAttributes(slug, attrId, codes);

            statics.verify(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper),
                    times(2));
        }
    }
}

@ExtendWith(MockitoExtension.class)
class CurrencyCacheOpsTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SearchRequestProperties properties;

    @Mock
    private CurrencyMapper currencyMapper;

    @InjectMocks
    private CurrencyCacheOps currencyCacheOps;

    /**
     * given: backend возвращает данные  
     * when:  вызываем loadAllCurrencies  
     * then:  запрос уходит в BaseMasterDataRequestService, результат мапится и отдается наружу
     */
    @Test
    void givenBackendResponse_whenLoadAllCurrencies_thenReturnMappedList() {
        // given
        String slug = "currency";
        String attrId = "currencyCode";

        when(properties.getSlugValueForCurrency()).thenReturn(slug);
        when(properties.getCurrencyAttributeId()).thenReturn(attrId);

        GetItemsSearchResponse resp = mock(GetItemsSearchResponse.class);
        when(baseMasterDataRequestService.requestDataByAttributes(slug, attrId, null))
                .thenReturn(resp);

        List<CurrencyDto> mapped = List.of(
                CurrencyDto.builder().currencyCode("USD").build(),
                CurrencyDto.builder().currencyCode("EUR").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     Mockito.mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() -> BaseMasterDataRequestService
                            .createResultWithAttribute(resp, currencyMapper))
                   .thenReturn(mapped);

            // when
            List<CurrencyDto> result = currencyCacheOps.loadAllCurrencies();

            // then
            assertEquals(mapped, result);

            verify(properties).getSlugValueForCurrency();
            verify(properties).getCurrencyAttributeId();
            verify(baseMasterDataRequestService)
                    .requestDataByAttributes(slug, attrId, null);
            statics.verify(() -> BaseMasterDataRequestService
                    .createResultWithAttribute(resp, currencyMapper));

            verifyNoMoreInteractions(baseMasterDataRequestService, properties, currencyMapper);
        }
    }

    /**
     * given: backend вернул пустой результат  
     * when:  вызываем loadAllCurrencies  
     * then:  наружу тоже уходит пустой список
     */
    @Test
    void givenEmptyBackendResponse_whenLoadAllCurrencies_thenReturnEmptyList() {
        // given
        String slug = "currency";
        String attrId = "currencyCode";

        when(properties.getSlugValueForCurrency()).thenReturn(slug);
        when(properties.getCurrencyAttributeId()).thenReturn(attrId);

        GetItemsSearchResponse resp = mock(GetItemsSearchResponse.class);
        when(baseMasterDataRequestService.requestDataByAttributes(slug, attrId, null))
                .thenReturn(resp);

        List<CurrencyDto> mapped = List.of(); // пустой список

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     Mockito.mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() -> BaseMasterDataRequestService
                            .createResultWithAttribute(resp, currencyMapper))
                   .thenReturn(mapped);

            // when
            List<CurrencyDto> result = currencyCacheOps.loadAllCurrencies();

            // then
            assertEquals(mapped, result);
        }
    }
}
```
