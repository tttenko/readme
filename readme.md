```java/**
@ExtendWith(MockitoExtension.class)
class CalendarDateServiceTest {

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    // ДОБАВЬ: зависимость, из-за которой сейчас NPE
    @Mock
    private CalendarHelper calendarHelper;

    @InjectMocks
    private CalendarDateService calendarDateService;

    @Test
    void givenDates_whenSearchByDates_thenUseCacheGetOrLoadService() {
        // given
        List<LocalDate> dateList = List.of(
            LocalDate.of(2026, 5, 25),
            LocalDate.of(2026, 5, 26)
        );

        List<String> expectedKeys = dateList.stream()
            .map(d -> d.format(DateTimeFormatter.ISO_DATE))
            .toList();

        List<CalendarDateDto> loaded = List.of(new CalendarDateDto(), new CalendarDateDto());

        when(cacheGetOrLoadService.<CalendarDateDto>fetchData(
            CalendarDateService.PROD_CALENDAR_DATE_BY_DATE,
            expectedKeys
        )).thenReturn(loaded);

        // when
        List<CalendarDateDto> result = calendarDateService.searchByDates(dateList);

        // then
        assertEquals(loaded, result);
        verify(cacheGetOrLoadService)
            .fetchData(CalendarDateService.PROD_CALENDAR_DATE_BY_DATE, expectedKeys);
        verifyNoMoreInteractions(cacheGetOrLoadService);
    }

    @Test
    void givenEmptyDates_whenSearchByDates_thenPassEmptyListToCache() {
        // given
        List<LocalDate> dateList = List.of();
        List<String> expectedKeys = List.of();
        List<CalendarDateDto> loaded = List.of(new CalendarDateDto());

        when(cacheGetOrLoadService.<CalendarDateDto>fetchData(
            CalendarDateService.PROD_CALENDAR_DATE_BY_DATE,
            expectedKeys
        )).thenReturn(loaded);

        // when
        List<CalendarDateDto> result = calendarDateService.searchByDates(dateList);

        // then
        assertEquals(loaded, result);
        verify(cacheGetOrLoadService)
            .fetchData(CalendarDateService.PROD_CALENDAR_DATE_BY_DATE, expectedKeys);
        verifyNoMoreInteractions(cacheGetOrLoadService);
    }
}
```
