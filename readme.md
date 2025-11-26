```java
@ExtendWith(MockitoExtension.class)
class LoaderCountryByCodeTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SearchRequestProperties properties;

    @Mock
    private CountryMapper countryMapper;

    @InjectMocks
    private LoaderCountryByCode loader;

    @Test
    void givenLoader_whenCacheName_thenReturnCountryByCode() {
        assertEquals(CountryService.COUNTRY_BY_CODE, loader.cacheName());
    }

    @Test
    void givenLoader_whenElementType_thenReturnCountryDtoClass() {
        assertEquals(CountryDto.class, loader.elementType());
    }

    @Test
    void givenCountryDto_whenExtractKey_thenReturnAlpha2Code() {
        CountryDto dto = new CountryDto();
        dto.setAlpha2Code("RU");

        String key = loader.extractKey(dto);

        assertEquals("RU", key);
    }

    @Test
    void givenEmptyKeys_whenFetchByKeys_thenReturnEmptyAndSkipBackend() {
        // given
        List<String> keys = List.of();

        // when
        List<CountryDto> result = loader.fetchByKeys(keys);

        // then
        assertTrue(result.isEmpty());
        verifyNoInteractions(baseMasterDataRequestService, properties, countryMapper);
    }

    @Test
    void givenCodes_whenFetchByKeys_thenDelegateToBackendAndMapResult() {
        // given
        List<String> codes = List.of("RU", "DE");
        String slug = "country";
        String attrId = "countryCode";

        when(properties.getSlugValueForCountry()).thenReturn(slug);
        when(properties.getAttributeIdCountry()).thenReturn(attrId);

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestDataByAttributes(slug, attrId, codes))
                .thenReturn(resp);

        List<CountryDto> mapped = List.of(
                CountryDto.builder().alpha2Code("RU").build(),
                CountryDto.builder().alpha2Code("DE").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() -> BaseMasterDataRequestService
                            .createResultWithAttribute(resp, countryMapper))
                    .thenReturn(mapped);

            // when
            List<CountryDto> result = loader.fetchByKeys(codes);

            // then
            assertEquals(mapped, result);

            verify(properties).getSlugValueForCountry();
            verify(properties).getAttributeIdCountry();
            verify(baseMasterDataRequestService, times(1))
                    .requestDataByAttributes(slug, attrId, codes);

            statics.verify(() -> BaseMasterDataRequestService
                            .createResultWithAttribute(resp, countryMapper),
                    times(1));

            verifyNoMoreInteractions(baseMasterDataRequestService, properties, countryMapper);
        }
    }

    @Test
    void givenTwoSequentialCalls_whenFetchByKeys_thenBackendInvokedTwice_noCaching() {
        // given
        List<String> codes = List.of("JP");
        String slug = "country";
        String attrId = "countryCode";

        when(properties.getSlugValueForCountry()).thenReturn(slug);
        when(properties.getAttributeIdCountry()).thenReturn(attrId);

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestDataByAttributes(slug, attrId, codes))
                .thenReturn(resp);

        List<CountryDto> mapped = List.of(
                CountryDto.builder().alpha2Code("JP").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() -> BaseMasterDataRequestService
                            .createResultWithAttribute(resp, countryMapper))
                    .thenReturn(mapped);

            // when
            List<CountryDto> r1 = loader.fetchByKeys(codes);
            List<CountryDto> r2 = loader.fetchByKeys(codes);

            // then
            assertEquals(mapped, r1);
            assertEquals(mapped, r2);

            verify(baseMasterDataRequestService, times(2))
                    .requestDataByAttributes(slug, attrId, codes);

            statics.verify(() -> BaseMasterDataRequestService
                            .createResultWithAttribute(resp, countryMapper),
                    times(2));
        }
    }
}
```
