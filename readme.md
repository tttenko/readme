```java
/**
     * Возвращает тип дня (рабочий/нерабочий) по списку дат.
     * Пример вызова: GET /api/v1/calendar/day-type?date=25.05.2026&date=26.05.2026
     */
    @GetExchange("/calendar/day-type")
    ResultObj<List<CalendDateDto>> searchProdCalendDatesByDate(
            @RequestParam("date") List<String> dates
    );
```
