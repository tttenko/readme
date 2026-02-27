```java/**
@ExtendWith(MockitoExtension.class)
class WorkDayCalculatorRangeLoadingTest {

    @Mock
    private CalendarHelper calendarHelper;

    private WorkDayCalculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new WorkDayCalculator(calendarHelper);
    }

    @Test
    void givenForwardAndNumDaysWithinPrefetchWindow_whenCalculate_thenLoadsOnceAndDoesNotReload() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 1);
        int numDays = 10; // при всех WORK => итераций будет 11, это точно внутри окна (32 даты)

        when(calendarHelper.getDates(any(LocalDate.class), eq(Duration.ofDays(31)), eq(true)))
                .thenAnswer(inv -> buildPrefetchMap(inv.getArgument(0), true));

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, true);

        // then
        assertThat(result).hasSize(numDays + 1);
        assertThat(result.get(0).getIsoDate()).isEqualTo("2026-01-01");
        assertThat(result.get(result.size() - 1).getIsoDate()).isEqualTo(start.plusDays(numDays).toString());

        // самое важное: ровно 1 загрузка окна
        verify(calendarHelper, times(1)).getDates(eq(start), eq(Duration.ofDays(31)), eq(true));
        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenForwardAndNumDaysCrossesPrefetchWindow_whenCalculate_thenReloadsExactlyWhenLeavingWindow() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 1);

        // Окно содержит 32 даты: start .. start+31 (inclusive).
        // Если все дни WORK, то цикл делает numDays+1 итераций.
        // Чтобы гарантированно выйти за окно в 32 даты, нужно numDays+1 > 32 => numDays >= 32.
        int numDays = 32; // итераций 33 => обязательно вторая загрузка

        when(calendarHelper.getDates(any(LocalDate.class), eq(Duration.ofDays(31)), eq(true)))
                .thenAnswer(inv -> buildPrefetchMap(inv.getArgument(0), true));

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, true);

        // then
        assertThat(result).hasSize(numDays + 1);
        assertThat(result)
                .extracting(CalendarDateDto::getIsoDate)
                .startsWith("2026-01-01")
                .endsWith(start.plusDays(numDays).toString());

        // Проверяем, что догрузка случилась ровно на границе:
        // первая загрузка: start (покрывает start..start+31)
        // вторая загрузка: start+32 (когда калькулятор дошел до даты, которой нет в мапе)
        ArgumentCaptor<LocalDate> startCaptor = ArgumentCaptor.forClass(LocalDate.class);
        verify(calendarHelper, times(2)).getDates(startCaptor.capture(), eq(Duration.ofDays(31)), eq(true));

        assertThat(startCaptor.getAllValues()).containsExactly(
                start,
                start.plusDays(32)
        );

        verifyNoMoreInteractions(calendarHelper);
    }

    @Test
    void givenBackwardAndNumDaysCrossesPrefetchWindow_whenCalculate_thenReloadsExactlyWhenLeavingWindow() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 31);
        int numDays = 32; // итераций 33 => нужна вторая загрузка

        when(calendarHelper.getDates(any(LocalDate.class), eq(Duration.ofDays(31)), eq(false)))
                .thenAnswer(inv -> buildPrefetchMap(inv.getArgument(0), false));

        // when
        List<CalendarDateDto> result = calculator.calculate(start, numDays, false);

        // then
        assertThat(result).hasSize(numDays + 1);
        assertThat(result.get(0).getIsoDate()).isEqualTo(start.toString());
        assertThat(result.get(result.size() - 1).getIsoDate()).isEqualTo(start.minusDays(numDays).toString());

        ArgumentCaptor<LocalDate> startCaptor = ArgumentCaptor.forClass(LocalDate.class);
        verify(calendarHelper, times(2)).getDates(startCaptor.capture(), eq(Duration.ofDays(31)), eq(false));

        // backward окно: start .. start-31 (inclusive). При выходе: start-32
        assertThat(startCaptor.getAllValues()).containsExactly(
                start,
                start.minusDays(32)
        );

        verifyNoMoreInteractions(calendarHelper);
    }

    /**
     * Возвращает "окно предзагрузки" на 31 день, которое в текущей реализации CalendarHelper
     * обычно превращается в 32 даты (inclusive границы): start..start±31.
     */
    private static Map<String, CalendarDateDto> buildPrefetchMap(LocalDate windowStart, boolean isForward) {
        Map<String, CalendarDateDto> map = new LinkedHashMap<>();
        for (int i = 0; i <= 31; i++) {
            LocalDate d = isForward ? windowStart.plusDays(i) : windowStart.minusDays(i);
            String iso = d.toString();
            map.put(iso, CalendarDateDto.builder()
                    .isoDate(iso)
                    .date(iso)
                    .dateShort(iso)
                    .dayType(DayType.WORK) // делаем все WORK, чтобы детерминировать количество итераций
                    .description("OK")
                    .build());
        }
        return map;
    }
}
```
