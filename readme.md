```java
@ExtendWith(MockitoExtension.class)
class CalendDateServiceTest {

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    @InjectMocks
    private CalendDateService calendDateService;

    @Test
    void givenDates_whenSearchByDates_thenUseCacheGetOrLoadService() {
        // given
        List<String> dates = List.of("25.05.2026", "26.05.2026");
        List<CalendDateDto> loaded = List.of(new CalendDateDto(), new CalendDateDto());

        when(cacheGetOrLoadService.fetchData(CalendDateService.PROD_CALEND_DATE_BY_DATE, dates))
                .thenReturn(loaded);

        // when
        List<CalendDateDto> result = calendDateService.searchByDates(dates);

        // then
        assertEquals(loaded, result);
        verify(cacheGetOrLoadService).fetchData(CalendDateService.PROD_CALEND_DATE_BY_DATE, dates);
        verifyNoMoreInteractions(cacheGetOrLoadService);
    }

    @Test
    void givenNullDates_whenSearchByDates_thenPassNullToCache() {
        // given
        List<CalendDateDto> loaded = List.of(new CalendDateDto());

        when(cacheGetOrLoadService.fetchData(
                eq(CalendDateService.PROD_CALEND_DATE_BY_DATE),
                ArgumentMatchers.<List<String>>isNull()
        )).thenReturn(loaded);

        // when
        List<CalendDateDto> result = calendDateService.searchByDates(null);

        // then
        assertEquals(loaded, result);
        verify(cacheGetOrLoadService).fetchData(
                eq(CalendDateService.PROD_CALEND_DATE_BY_DATE),
                ArgumentMatchers.<List<String>>isNull()
        );
        verifyNoMoreInteractions(cacheGetOrLoadService);
    }

    @Test
    void givenEmptyDates_whenSearchByDates_thenPassEmptyListToCache() {
        // given
        List<String> dates = List.of();
        List<CalendDateDto> loaded = List.of(new CalendDateDto());

        when(cacheGetOrLoadService.fetchData(CalendDateService.PROD_CALEND_DATE_BY_DATE, dates))
                .thenReturn(loaded);

        // when
        List<CalendDateDto> result = calendDateService.searchByDates(dates);

        // then
        assertEquals(loaded, result);
        verify(cacheGetOrLoadService).fetchData(CalendDateService.PROD_CALEND_DATE_BY_DATE, dates);
        verifyNoMoreInteractions(cacheGetOrLoadService);
    }
}
```
