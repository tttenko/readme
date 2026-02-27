```java/**
@ExtendWith(MockitoExtension.class)
class WorkDayCalculatorTest {

    @Mock
    private CalendarHelper calendarHelper;

    private WorkDayCalculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new WorkDayCalculator(calendarHelper);
    }

    @Test
    void givenForwardAndCalendarContainsWorkAndNonWorkDays_whenCalculate_thenReturnsAllDaysUntilNumDaysPlusOneWorkDaysReached() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 2; // при текущем while(dayCount>-1) => доберет 3 рабочих дня

        // Сценарий:
        // 30 WORK  (work#1)
        // 31 COMMON
        // 01 COMMON
        // 02 WORK  (work#2)
        // 03 WORK  (work#3) -> после него dayCount станет -1 и цикл завершится
        Map<String, CalendarDateDto> map = new LinkedHashMap<>();
        map.put(start.toString(), dto(start, DayType.WORK));
        map.put(start.plusDays(1).toString(), dto(start.plusDays(1), DayType.COMMON));
        map.put(start.plusDays(2).toString(), dto(start.plusDays(2), DayType.COMMON));
        map.put(start.plusDays(3).toString(), dto(start.plusDays(3), DayType.WORK));
        map.put(start.plusDays(4).toString(), dto(start.plusDays(4), DayType.WORK));

        when(calendarHelper.getDates(eq(start), any(Duration.class), eq(true))).thenReturn(map);

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, true);

        // then
        assertThat(result).hasSize(5);
        assertThat(result)
                .extracting(CalendarDateDto::getIsoDate)
                .containsExactly(
                        "2026-01-30",
                        "2026-01-31",
                        "2026-02-01",
                        "2026-02-02",
                        "2026-02-03"
                );

        // WORK дни внутри результата = 3 (numDays + 1)
        long workCount = result.stream().filter(d -> d.getDayType().equals(DayType.WORK)).count();
        assertThat(workCount).isEqualTo(3);

        verify(calendarHelper, times(1)).getDates(eq(start), any(Duration.class), eq(true));
        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenBackwardAndCalendarContainsWorkAndNonWorkDays_whenCalculate_thenReturnsAllDaysUntilNumDaysPlusOneWorkDaysReached() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 1; // при текущем while(dayCount>-1) => доберет 2 рабочих дня

        // Сценарий назад:
        // 30 WORK (work#1)
        // 29 COMMON
        // 28 WORK (work#2) -> после него dayCount станет -1 и цикл завершится
        Map<String, CalendarDateDto> map = new LinkedHashMap<>();
        map.put(start.toString(), dto(start, DayType.WORK));
        map.put(start.minusDays(1).toString(), dto(start.minusDays(1), DayType.COMMON));
        map.put(start.minusDays(2).toString(), dto(start.minusDays(2), DayType.WORK));

        when(calendarHelper.getDates(eq(start), any(Duration.class), eq(false))).thenReturn(map);

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

        long workCount = result.stream().filter(d -> d.getDayType().equals(DayType.WORK)).count();
        assertThat(workCount).isEqualTo(2);

        verify(calendarHelper, times(1)).getDates(eq(start), any(Duration.class), eq(false));
        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenForwardAndPrefetchMissingNextDay_whenCalculate_thenFetchesAgainForNextCurrentDate() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 1; // при текущем while(dayCount>-1) => доберет 2 рабочих дня

        // 1-й вызов: есть только стартовая дата (WORK)
        Map<String, CalendarDateDto> map1 = new LinkedHashMap<>();
        map1.put(start.toString(), dto(start, DayType.WORK));

        // 2-й вызов (на 31): добавим 31 COMMON и 01 WORK (чтобы добрать 2-й рабочий день)
        Map<String, CalendarDateDto> map2 = new LinkedHashMap<>();
        map2.put(start.plusDays(1).toString(), dto(start.plusDays(1), DayType.COMMON));
        map2.put(start.plusDays(2).toString(), dto(start.plusDays(2), DayType.WORK));

        when(calendarHelper.getDates(eq(start), any(Duration.class), eq(true))).thenReturn(map1);
        when(calendarHelper.getDates(eq(start.plusDays(1)), any(Duration.class), eq(true))).thenReturn(map2);

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, true);

        // then
        assertThat(result)
                .extracting(CalendarDateDto::getIsoDate)
                .containsExactly(
                        "2026-01-30", // WORK (work#1)
                        "2026-01-31", // COMMON
                        "2026-02-01"  // WORK (work#2) -> stop
                );

        verify(calendarHelper, times(1)).getDates(eq(start), any(Duration.class), eq(true));
        verify(calendarHelper, times(1)).getDates(eq(start.plusDays(1)), any(Duration.class), eq(true));
        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenNumDaysZeroAndStartIsWork_whenCalculate_thenStopsAfterOneWorkDayAndReturnsSingleDate() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 0; // while(dayCount>-1) => нужно 1 WORK день, чтобы dayCount стал -1

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
                .isoDate(iso)   // ключ в dateMap
                .date(iso)
                .dateShort(iso)
                .dayType(dayType)
                .description("OK")
                .build();
    }
}
```
