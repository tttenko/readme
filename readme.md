```java/**
@ExtendWith(MockitoExtension.class)
class CalculatorFactoryTest {

    @Mock
    private CalendarHelper calendarHelper;

    @Test
    void givenWorkDayType_whenGetInstance_thenReturnsWorkDayCalculator() {
        // given
        CalculatorFactory factory = new CalculatorFactory(calendarHelper);

        // when
        DayRangeCalculator calculator = factory.getInstance(DayType.WORK);

        // then
        assertThat(calculator).isInstanceOf(WorkDayCalculator.class);
    }

    @Test
    void givenCommonDayType_whenGetInstance_thenReturnsCommonDayCalculator() {
        // given
        CalculatorFactory factory = new CalculatorFactory(calendarHelper);

        // when
        DayRangeCalculator calculator = factory.getInstance(DayType.COMMON);

        // then
        assertThat(calculator).isInstanceOf(CommonDayCalculator.class);
    }

    @Test
    void givenUndefinedDayType_whenGetInstance_thenReturnsCommonDayCalculator() {
        // given
        CalculatorFactory factory = new CalculatorFactory(calendarHelper);

        // when
        DayRangeCalculator calculator = factory.getInstance(DayType.UNDEFINED);

        // then
        assertThat(calculator).isInstanceOf(CommonDayCalculator.class);
    }
}


@ExtendWith(MockitoExtension.class)
class CalendarDateServiceTest {

    @Mock
    private CalculatorFactory factory;

    @Mock
    private CalendarHelper calendarHelper;

    @Mock
    private DayRangeCalculator calculator;

    private CalendarDateService service;

    @BeforeEach
    void setUp() {
        service = new CalendarDateService(factory, calendarHelper);
    }

    @Test
    void givenParams_whenBuildRange_thenUsesFactoryCalculatorAndDelegatesCalculate() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);
        int numDays = 3;
        boolean isForward = true;
        DayType dayType = DayType.WORK;

        List<CalendarDateDto> expected = List.of(
                CalendarDateDto.builder().isoDate("2026-01-30").dayType(DayType.WORK).description("OK").build()
        );

        when(factory.getInstance(dayType)).thenReturn(calculator);
        when(calculator.calculate(start, numDays, isForward)).thenReturn(expected);

        // when
        List<CalendarDateDto> result = service.buildRange(start, numDays, isForward, dayType);

        // then
        assertThat(result).isSameAs(expected);

        verify(factory, times(1)).getInstance(dayType);
        verify(calculator, times(1)).calculate(start, numDays, isForward);
        verifyNoInteractions(calendarHelper);
    }

    @Test
    void givenDates_whenSearchByDates_thenDelegatesToCalendarHelper() {
        // given
        List<LocalDate> dates = List.of(
                LocalDate.of(2026, 1, 30),
                LocalDate.of(2026, 2, 1)
        );

        List<CalendarDateDto> expected = List.of(
                CalendarDateDto.builder().isoDate("2026-01-30").dayType(DayType.WORK).description("OK").build()
        );

        when(calendarHelper.searchByDates(dates)).thenReturn(expected);

        // when
        List<CalendarDateDto> result = service.searchByDates(dates);

        // then
        assertThat(result).isSameAs(expected);

        verify(calendarHelper, times(1)).searchByDates(dates);
        verifyNoInteractions(factory);
    }

    @Test
    void givenDifferentDayTypes_whenBuildRange_thenPassesSameDayTypeToFactory() {
        // given
        LocalDate start = LocalDate.of(2026, 1, 30);

        when(factory.getInstance(any())).thenReturn(calculator);
        when(calculator.calculate(any(), anyInt(), anyBoolean())).thenReturn(List.of());

        // when
        service.buildRange(start, 1, true, DayType.WORK);
        service.buildRange(start, 1, true, DayType.COMMON);
        service.buildRange(start, 1, true, DayType.UNDEFINED);

        // then
        verify(factory, times(1)).getInstance(DayType.WORK);
        verify(factory, times(1)).getInstance(DayType.COMMON);
        verify(factory, times(1)).getInstance(DayType.UNDEFINED);

        verify(calculator, times(3)).calculate(eq(start), eq(1), eq(true));
        verifyNoInteractions(calendarHelper);
    }
}
```
