```java/**
public interface CalendarDateController {

    String DATE_REGEX = "\\d{4}-\\d{2}-\\d{2}$"; // если где-то всё же нужен regex

    @GetMapping("/day-type")
    ResultObj<List<CalendarDateDto>> searchProdCalendarDatesByDate(
        @ArraySchema(
            schema = @Schema(
                description = "Дата в формате yyyy-MM-dd (ISO-8601)",
                example = "2026-05-05"
            )
        )
        @RequestParam("date")
        @NotEmpty(message = "Параметр date должен содержать хотя бы одну дату")
        @org.springframework.format.annotation.DateTimeFormat(iso = org.springframework.format.annotation.DateTimeFormat.ISO.DATE)
        List<@jakarta.validation.constraints.NotNull(message = "date не должен быть null") LocalDate> date
    );
}

@Override
public ResultObj<List<CalendarDateDto>> searchProdCalendarDatesByDate(
    @RequestParam("date")
    @NotEmpty(message = "Параметр date должен содержать хотя бы одну дату")
    @DateTimeFormat(iso = DateTimeFormat.ISO.DATE)
    List<@NotNull LocalDate> date
) {
    return getSuccessResponse(prodCalendDateService.searchByDates(date));
}
```
