```java
/**
 * Возвращает диапазон календарных дат.
 *
 * <p>Список всегда отсортирован по возрастанию даты.</p>
 * <ul>
 *   <li>{@code daysClassification=all} — ровно {@code numDays} календарных дней от {@code date}.</li>
 *   <li>{@code daysClassification=work} — диапазон от {@code date} до даты,
 *       на которой набралось {@code numDays} рабочих дней.</li>
 * </ul>
 *
 * @param date стартовая дата (формат запроса {@code dd.MM.yyyy})
 * @param numDays количество дней (минимум 1)
 * @param isForward направление расчёта: {@code true} — вперёд, {@code false} — назад
 * @param daysClassification режим расчёта: {@code all} или {@code work}
 */


 /**
 * Реализация эндпоинта получения диапазона дат.
 *
 * <p>Аннотации параметров продублированы из интерфейса, чтобы Spring MVC/валидация
 * гарантированно применялись на методе реализации.</p>
 */
@Override
@GetMapping("/range")
public ResultObj<List<CalendarRangeItemDto>> calendarRange(
        @RequestParam("date")
        @NotNull(message = "date не должен быть пустым")
        @DateTimeFormat(pattern = "dd.MM.yyyy")
        LocalDate date,

        @RequestParam("numDays")
        @Min(value = 1, message = "numDays должен быть >= 1")
        int numDays,

        @RequestParam(value = "isForward", required = false, defaultValue = "true")
        boolean isForward,

        @RequestParam(value = "daysClassification", required = false, defaultValue = "all")
        String daysClassification
) {
    List<CalendarRangeItemDto> items =
            calendarRangeService.buildRange(date, numDays, isForward, daysClassification);

    return getSuccessResponse(items);
}
```
