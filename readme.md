```java

public class MissingCalendarDataException extends RuntimeException {

    private static final DateTimeFormatter DDMMYYYY = DateTimeFormatter.ofPattern("dd.MM.yyyy");

    private final LocalDate date;

    public MissingCalendarDataException(LocalDate date) {
        super("No calendar data from MD for date: " + format(date));
        this.date = date;
    }

    public LocalDate getDate() {
        return date;
    }

    public String getDateShort() {
        return format(date);
    }

    private static String format(LocalDate d) {
        return d == null ? "null" : d.format(DDMMYYYY);
    }
}

```
