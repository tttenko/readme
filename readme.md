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
```
