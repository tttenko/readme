```java/**
@ExtendWith(MockitoExtension.class)
class CommonDayCalculatorTest {

    @Mock
    private CalendarHelper calendarHelper;

    private static final Duration ONE_MONTH_DURATION = Duration.of(1, ChronoUnit.MONTHS);

    @Test
    void givenForwardRangeAndMapHasAllNeededDates_whenCalculate_thenReturnsNumDaysPlusOneInAscendingOrderAndFetchesOnce() {
        // given
        CommonDayCalculator calculator = new CommonDayCalculator(calendarHelper);

        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 2; // из-за while(dayCount > -1) ожидаем 3 элемента

        Map<String, CalendarDateDto> monthMap = new LinkedHashMap<>();
        monthMap.put(start.toString(), dto(start, DayType.WORK));
        monthMap.put(start.plusDays(1).toString(), dto(start.plusDays(1), DayType.COMMON));
        monthMap.put(start.plusDays(2).toString(), dto(start.plusDays(2), DayType.COMMON));

        when(calendarHelper.getDates(start, ONE_MONTH_DURATION, true)).thenReturn(monthMap);

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, true);

        // then
        assertThat(result).hasSize(3);
        assertThat(result)
                .extracting(CalendarDateDto::getIsoDate)
                .containsExactly(
                        "2026-01-30",
                        "2026-01-31",
                        "2026-02-01"
                );

        verify(calendarHelper, times(1)).getDates(start, ONE_MONTH_DURATION, true);
        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenBackwardRangeAndMapHasAllNeededDates_whenCalculate_thenReturnsNumDaysPlusOneInDescendingOrderAndFetchesOnce() {
        // given
        CommonDayCalculator calculator = new CommonDayCalculator(calendarHelper);

        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 2; // ожидаем 3 элемента

        Map<String, CalendarDateDto> monthMap = new LinkedHashMap<>();
        monthMap.put(start.toString(), dto(start, DayType.WORK));
        monthMap.put(start.minusDays(1).toString(), dto(start.minusDays(1), DayType.COMMON));
        monthMap.put(start.minusDays(2).toString(), dto(start.minusDays(2), DayType.COMMON));

        when(calendarHelper.getDates(start, ONE_MONTH_DURATION, false)).thenReturn(monthMap);

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, false);

        // then
        assertThat(result).hasSize(3);
        assertThat(result)
                .extracting(CalendarDateDto::getIsoDate)
                .containsExactly(
                        "2026-01-30",
                        "2026-01-29",
                        "2026-01-28"
                );

        verify(calendarHelper, times(1)).getDates(start, ONE_MONTH_DURATION, false);
        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenForwardRangeWithCacheMissOnNextDay_whenCalculate_thenFetchesAgainForNextCurrentDate() {
        // given
        CommonDayCalculator calculator = new CommonDayCalculator(calendarHelper);

        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 2; // итоговый размер = 3

        Map<String, CalendarDateDto> map1 = new LinkedHashMap<>();
        map1.put(start.toString(), dto(start, DayType.WORK)); // только старт

        Map<String, CalendarDateDto> map2 = new LinkedHashMap<>();
        map2.put(start.plusDays(1).toString(), dto(start.plusDays(1), DayType.COMMON));
        map2.put(start.plusDays(2).toString(), dto(start.plusDays(2), DayType.COMMON));

        when(calendarHelper.getDates(start, ONE_MONTH_DURATION, true)).thenReturn(map1);
        when(calendarHelper.getDates(start.plusDays(1), ONE_MONTH_DURATION, true)).thenReturn(map2);

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, true);

        // then
        assertThat(result).hasSize(3);
        assertThat(result)
                .extracting(CalendarDateDto::getIsoDate)
                .containsExactly(
                        "2026-01-30",
                        "2026-01-31",
                        "2026-02-01"
                );

        verify(calendarHelper, times(1)).getDates(start, ONE_MONTH_DURATION, true);
        verify(calendarHelper, times(1)).getDates(start.plusDays(1), ONE_MONTH_DURATION, true);
        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenNumDaysZero_whenCalculate_thenReturnsSingleStartDate() {
        // given
        CommonDayCalculator calculator = new CommonDayCalculator(calendarHelper);

        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 0; // вернётся 1 элемент

        Map<String, CalendarDateDto> monthMap = new LinkedHashMap<>();
        monthMap.put(start.toString(), dto(start, DayType.WORK));

        when(calendarHelper.getDates(start, ONE_MONTH_DURATION, true)).thenReturn(monthMap);

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, true);

        // then
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getIsoDate()).isEqualTo("2026-01-30");

        verify(calendarHelper, times(1)).getDates(start, ONE_MONTH_DURATION, true);
        verifyNoMoreInteractions(calendarHelper);
    }

    private static CalendarDateDto dto(LocalDate date, DayType dayType) {
        String iso = date.toString(); // yyyy-MM-dd
        return CalendarDateDto.builder()
                .isoDate(iso)     // ключ, по которому кладём в Map и по которому достаём
                .date(iso)
                .dateShort(iso)
                .dayType(dayType)
                .description("OK")
                .build();
    }
}
```
