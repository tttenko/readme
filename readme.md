```java/**
@Test
@DisplayName("test GET {host}/api/v1/calendar/day-type/range (default params)")
void calendarRange_defaultParamsTest() throws Exception {
    // given
    LocalDate date = LocalDate.of(2026, 1, 1);
    int numDays = 2;

    when(calendarDateService.buildRange(date, numDays, true, DayType.COMMON))
            .thenReturn(Collections.emptyList());

    // when + then
    MvcTestUtils.checkResult(
            MvcTestUtils.performGetOk(
                    mockMvc,
                    "/api/v1/calendar/day-type/range?date=2026-01-01&numDays=2"
            ),
            0
    );

    verify(calendarDateService).buildRange(date, numDays, true, DayType.COMMON);
}

@Test
@DisplayName("test GET {host}/api/v1/calendar/day-type/range (all params)")
void calendarRange_allParamsTest() throws Exception {
    // given
    LocalDate date = LocalDate.of(2026, 1, 1);
    int numDays = 5;

    when(calendarDateService.buildRange(date, numDays, false, DayType.WORK))
            .thenReturn(Collections.emptyList());

    // when + then
    MvcTestUtils.checkResult(
            MvcTestUtils.performGetOk(
                    mockMvc,
                    "/api/v1/calendar/day-type/range?date=2026-01-01&numDays=5&isForward=false&dayType=WORK"
            ),
            0
    );

    verify(calendarDateService).buildRange(date, numDays, false, DayType.WORK);
}
```
