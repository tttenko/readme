```java/**
@ExtendWith(MockitoExtension.class)
class CommonDayCalculatorTest {

    @Mock
    private CalendarHelper calendarHelper;

    private CommonDayCalculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new CommonDayCalculator(calendarHelper);
    }

    @Test
    void givenForwardAndCalendarHasAllDates_whenCalculate_thenReturnsNumDaysPlusOneAndCallsGetDatesOnce() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 2; // текущая реализация while(dayCount > -1) => вернет 3 элемента

        Map<String, CalendarDateDto> map = new LinkedHashMap<>();
        map.put(start.toString(), dto(start, DayType.WORK));
        map.put(start.plusDays(1).toString(), dto(start.plusDays(1), DayType.COMMON));
        map.put(start.plusDays(2).toString(), dto(start.plusDays(2), DayType.COMMON));

        when(calendarHelper.getDates(eq(start), any(Duration.class), eq(true))).thenReturn(map);

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, true);

        // then
        assertThat(result).hasSize(3);
        assertThat(result)
                .extracting(CalendarDateDto::getIsoDate)
                .containsExactly("2026-01-30", "2026-01-31", "2026-02-01");

        verify(calendarHelper, times(1)).getDates(eq(start), any(Duration.class), eq(true));
        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenBackwardAndCalendarHasAllDates_whenCalculate_thenReturnsNumDaysPlusOneAndCallsGetDatesOnce() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 2; // => 3 элемента

        Map<String, CalendarDateDto> map = new LinkedHashMap<>();
        map.put(start.toString(), dto(start, DayType.WORK));
        map.put(start.minusDays(1).toString(), dto(start.minusDays(1), DayType.COMMON));
        map.put(start.minusDays(2).toString(), dto(start.minusDays(2), DayType.COMMON));

        when(calendarHelper.getDates(eq(start), any(Duration.class), eq(false))).thenReturn(map);

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, false);

        // then
        assertThat(result).hasSize(3);
        assertThat(result)
                .extracting(CalendarDateDto::getIsoDate)
                .containsExactly("2026-01-30", "2026-01-29", "2026-01-28");

        verify(calendarHelper, times(1)).getDates(eq(start), any(Duration.class), eq(false));
        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenForwardAndPrefetchDoesNotContainNextDay_whenCalculate_thenCallsGetDatesAgainForNextCurrentDate() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 2; // => 3 элемента

        Map<String, CalendarDateDto> map1 = new LinkedHashMap<>();
        map1.put(start.toString(), dto(start, DayType.WORK)); // только старт

        Map<String, CalendarDateDto> map2 = new LinkedHashMap<>();
        map2.put(start.plusDays(1).toString(), dto(start.plusDays(1), DayType.COMMON));
        map2.put(start.plusDays(2).toString(), dto(start.plusDays(2), DayType.COMMON));

        when(calendarHelper.getDates(eq(start), any(Duration.class), eq(true))).thenReturn(map1);
        when(calendarHelper.getDates(eq(start.plusDays(1)), any(Duration.class), eq(true))).thenReturn(map2);

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, true);

        // then
        assertThat(result).hasSize(3);
        assertThat(result)
                .extracting(CalendarDateDto::getIsoDate)
                .containsExactly("2026-01-30", "2026-01-31", "2026-02-01");

        verify(calendarHelper, times(1)).getDates(eq(start), any(Duration.class), eq(true));
        verify(calendarHelper, times(1)).getDates(eq(start.plusDays(1)), any(Duration.class), eq(true));
        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenNumDaysZero_whenCalculate_thenReturnsSingleStartDate() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 0; // => 1 элемент

        Map<String, CalendarDateDto> map = new LinkedHashMap<>();
        map.put(start.toString(), dto(start, DayType.WORK));

        when(calendarHelper.getDates(eq(start), any(Duration.class), eq(true))).thenReturn(map);

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, true);

        // then
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getIsoDate()).isEqualTo("2026-01-30");

        verify(calendarHelper, times(1)).getDates(eq(start), any(Duration.class), eq(true));
        verifyNoMoreInteractions(calendarHelper);
    }

    private static CalendarDateDto dto(LocalDate date, DayType dayType) {
        String iso = date.toString(); // yyyy-MM-dd
        return CalendarDateDto.builder()
                .isoDate(iso)   // ВАЖНО: ключ для map.put(dto.getIsoDate(), dto)
                .date(iso)
                .dateShort(iso)
                .dayType(dayType)
                .description("OK")
                .build();
    }
}
```
