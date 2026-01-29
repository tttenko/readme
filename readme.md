```java
@Override
    public ResultObj<List<CalendDateDto>> searchProdCalendDatesByDate(
            @RequestParam("date")
            @NotEmpty(message = "Параметр date должен содержать хотя бы одну дату")
            List<
                @NotBlank(message = "date не должен быть пустым")
                @Pattern(regexp = DATE_REGEX, message = "Дата должна быть в формате dd.MM.yyyy")
                String
            > date
    ) {
        return getSuccessResponse(prodCalendDateService.searchByDates(date));
    }
```
