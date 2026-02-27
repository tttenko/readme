```java/**
@ExtendWith(MockitoExtension.class)
class CalendarHelperTest {

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    private CalendarHelper calendarHelper;

    @BeforeEach
    void setUp() {
        calendarHelper = new CalendarHelper(cacheGetOrLoadService);
    }

    @Test
    void givenForwardRangeWithMissingDates_whenGetDates_thenAddsUndefinedForMissingDates() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        Duration duration = Duration.ofDays(2); // 2026-01-30, 2026-01-31, 2026-02-01 (inclusive)

        CalendarDateDto d30 = CalendarDateDto.builder()
                .dateShort("2026-01-30")
                .date("2026-01-30")
                .dayType(DayType.WORK)
                .description("OK")
                .build();

        // В DayType нет NONWORK — используем COMMON как "не WORK" (обычный/нерабочий/любой не-рабочий тип)
        CalendarDateDto d01 = CalendarDateDto.builder()
                .dateShort("2026-02-01")
                .date("2026-02-01")
                .dayType(DayType.COMMON)
                .description("OK")
                .build();

        when(cacheGetOrLoadService.fetchData(any(), any()))
                .thenReturn(List.of(d30, d01)); // 2026-01-31 отсутствует

        // when
        Map<String, CalendarDateDto> result = calendarHelper.getDates(start, duration, true);

        // then
        assertThat(result).hasSize(3);
        assertThat(result).containsKeys("2026-01-30", "2026-01-31", "2026-02-01");

        assertThat(result.get("2026-01-30").getDayType()).isEqualTo(DayType.WORK);
        assertThat(result.get("2026-02-01").getDayType()).isEqualTo(DayType.COMMON);

        CalendarDateDto missing = result.get("2026-01-31");
        assertThat(missing.getDayType()).isEqualTo(DayType.UNDEFINED);
        assertThat(missing.getDescription()).isEqualTo("Нет информации о дне");

        // проверяем, какие ключи ушли в кэш
        @SuppressWarnings("unchecked")
        ArgumentCaptor<List<String>> keysCaptor = ArgumentCaptor.forClass(List.class);
        verify(cacheGetOrLoadService, times(1)).fetchData(any(), keysCaptor.capture());

        assertThat(keysCaptor.getValue()).containsExactly(
                "2026-01-30",
                "2026-01-31",
                "2026-02-01"
        );

        verifyNoMoreInteractions(cacheGetOrLoadService);
    }

    @Test
    void givenBackwardRange_whenGetDates_thenRequestsIsoKeysInBackwardOrderAndFillsMissing() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        Duration duration = Duration.ofDays(2); // ожидаем: 30, 29, 28 (inclusive) при isForward=false

        CalendarDateDto d30 = CalendarDateDto.builder()
                .dateShort("2026-01-30")
                .date("2026-01-30")
                .dayType(DayType.WORK)
                .description("OK")
                .build();

        when(cacheGetOrLoadService.fetchData(any(), any()))
                .thenReturn(List.of(d30)); // 29 и 28 будут UNDEFINED

        // when
        Map<String, CalendarDateDto> result = calendarHelper.getDates(start, duration, false);

        // then
        assertThat(result).hasSize(3);
        assertThat(result).containsKeys("2026-01-30", "2026-01-29", "2026-01-28");

        assertThat(result.get("2026-01-30").getDayType()).isEqualTo(DayType.WORK);
        assertThat(result.get("2026-01-29").getDayType()).isEqualTo(DayType.UNDEFINED);
        assertThat(result.get("2026-01-28").getDayType()).isEqualTo(DayType.UNDEFINED);

        @SuppressWarnings("unchecked")
        ArgumentCaptor<List<String>> keysCaptor = ArgumentCaptor.forClass(List.class);
        verify(cacheGetOrLoadService, times(1)).fetchData(any(), keysCaptor.capture());

        assertThat(keysCaptor.getValue()).containsExactly(
                "2026-01-30",
                "2026-01-29",
                "2026-01-28"
        );

        verifyNoMoreInteractions(cacheGetOrLoadService);
    }

    @Test
    void givenDatesList_whenSearchByDates_thenPassesIsoDateKeysToCache() {
        // given
        List<LocalDate> dates = List.of(
                LocalDate.of(2026, 1, 30),
                LocalDate.of(2026, 2, 1)
        );

        CalendarDateDto d30 = CalendarDateDto.builder()
                .dateShort("2026-01-30")
                .date("2026-01-30")
                .dayType(DayType.WORK)
                .description("OK")
                .build();

        when(cacheGetOrLoadService.fetchData(any(), any()))
                .thenReturn(List.of(d30));

        // when
        List<CalendarDateDto> result = calendarHelper.searchByDates(dates);

        // then
        assertThat(result).containsExactly(d30);

        @SuppressWarnings("unchecked")
        ArgumentCaptor<List<String>> keysCaptor = ArgumentCaptor.forClass(List.class);
        verify(cacheGetOrLoadService, times(1)).fetchData(any(), keysCaptor.capture());

        assertThat(keysCaptor.getValue()).containsExactly(
                "2026-01-30",
                "2026-02-01"
        );

        verifyNoMoreInteractions(cacheGetOrLoadService);
    }

    @Test
    void givenStartAndDuration_whenGetDateRangeForward_thenReturnsInclusiveRange() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        Duration duration = Duration.ofDays(2);

        // when
        List<LocalDate> dates = CalendarHelper.getDateRangeForward(start, duration);

        // then
        assertThat(dates).containsExactly(
                LocalDate.of(2026, 1, 30),
                LocalDate.of(2026, 1, 31),
                LocalDate.of(2026, 2, 1)
        );
    }

    @Test
    void givenStartAndDuration_whenGetDateRangeBackward_thenReturnsInclusiveRange() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        Duration duration = Duration.ofDays(2);

        // when
        List<LocalDate> dates = CalendarHelper.getDateRangeBackward(start, duration);

        // then
        assertThat(dates).containsExactly(
                LocalDate.of(2026, 1, 30),
                LocalDate.of(2026, 1, 29),
                LocalDate.of(2026, 1, 28)
        );
    }

    /**
     * Если у вас доступна константа PROD_CALENDAR_DATE_BY_DATE, то вместо any() можно сделать:
     * verify(cacheGetOrLoadService).fetchData(eq(PROD_CALENDAR_DATE_BY_DATE), keysCaptor.capture());
     */
}
```
