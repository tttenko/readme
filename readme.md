```java
@ExtendWith(MockitoExtension.class)
class CountryCacheOpsTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SearchRequestProperties properties;

    @Mock
    private CountryMapper countryMapper;

    @InjectMocks
    private CountryCacheOps countryCacheOps;

    @Test
    void givenBackendResponse_whenLoadAllCountries_thenReturnMappedList() {
        // given
        String slug = "country";
        String attrId = "countryCode";

        when(properties.getSlugValueForCountry()).thenReturn(slug);
        when(properties.getAttributeIdCountry()).thenReturn(attrId);

        GetItemsSearchResponse resp = mock(GetItemsSearchResponse.class);
        when(baseMasterDataRequestService.requestDataByAttributes(slug, attrId, null))
                .thenReturn(resp);

        List<CountryDto> mapped = List.of(
                CountryDto.builder().alpha2Code("RU").build(),
                CountryDto.builder().alpha2Code("DE").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     Mockito.mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() -> BaseMasterDataRequestService
                            .createResultWithAttribute(resp, countryMapper))
                    .thenReturn(mapped);

            // when
            List<CountryDto> result = countryCacheOps.loadAllCountries();

            // then
            assertEquals(mapped, result);

            verify(properties).getSlugValueForCountry();
            verify(properties).getAttributeIdCountry();
            verify(baseMasterDataRequestService)
                    .requestDataByAttributes(slug, attrId, null);
            statics.verify(() -> BaseMasterDataRequestService
                    .createResultWithAttribute(resp, countryMapper));

            verifyNoMoreInteractions(baseMasterDataRequestService, properties, countryMapper);
        }
    }

    @Test
    void givenEmptyBackendResponse_whenLoadAllCountries_thenReturnEmptyList() {
        // given
        String slug = "country";
        String attrId = "countryCode";

        when(properties.getSlugValueForCountry()).thenReturn(slug);
        when(properties.getAttributeIdCountry()).thenReturn(attrId);

        GetItemsSearchResponse resp = mock(GetItemsSearchResponse.class);
        when(baseMasterDataRequestService.requestDataByAttributes(slug, attrId, null))
                .thenReturn(resp);

        List<CountryDto> mapped = List.of(); // пустой список

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     Mockito.mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() -> BaseMasterDataRequestService
                            .createResultWithAttribute(resp, countryMapper))
                    .thenReturn(mapped);

            // when
            List<CountryDto> result = countryCacheOps.loadAllCountries();

            // then
            assertEquals(mapped, result);

            verify(properties).getSlugValueForCountry();
            verify(properties).getAttributeIdCountry();
            verify(baseMasterDataRequestService)
                    .requestDataByAttributes(slug, attrId, null);
            statics.verify(() -> BaseMasterDataRequestService
                    .createResultWithAttribute(resp, countryMapper));

            verifyNoMoreInteractions(baseMasterDataRequestService, properties, countryMapper);
        }
    }
}
```
