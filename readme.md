```java
@ExtendWith(MockitoExtension.class)
class CountryServiceTest {

    @Mock
    private CountryCacheOps countryCacheOps;

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    @InjectMocks
    private CountryService countryService;

    @Test
    void givenCodes_whenSearchCountriesByCode_thenUseCacheGetOrLoadService() {
        // given
        List<String> codes = List.of("RU", "DE");
        List<CountryDto> loaded = List.of(
                new CountryDto(),
                new CountryDto()
        );

        when(cacheGetOrLoadService.fetchData(CountryService.COUNTRY_BY_CODE, codes))
                .thenReturn(loaded);

        // when
        List<CountryDto> result = countryService.searchCountriesByCode(codes);

        // then
        assertEquals(loaded, result);
        verify(cacheGetOrLoadService)
                .fetchData(CountryService.COUNTRY_BY_CODE, codes);
        verifyNoInteractions(countryCacheOps);
    }

    @Test
    void whenGetAllCountries_thenLoadAllFromCountryCacheOps() {
        // given
        List<CountryDto> all = List.of(
                new CountryDto(),
                new CountryDto()
        );

        when(countryCacheOps.loadAllCountries()).thenReturn(all);

        // when
        List<CountryDto> result = countryService.getAllCountries();

        // then
        assertEquals(all, result);
        verify(countryCacheOps).loadAllCountries();
        verifyNoInteractions(cacheGetOrLoadService);
    }
}
```
