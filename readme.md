```java
/**
 * Ищет информацию о типе дня по заданной дате (производственный календарь).
 *
 * @param date дата в формате dd.MM.yyyy (например, "25.05.2026")
 * @return {@link ResultObj} со списком найденных {@link CalendDateDto}
 */
@GetExchange("/info/prod_calend_date/{date}")
ResultObj<List<CalendDateDto>> searchProdCalendDateByDate(
        @PathVariable("date") String date
);
```
