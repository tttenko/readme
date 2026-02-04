```java

@Component
@RequiredArgsConstructor
public class CalendarDataProvider {

    private static final DateTimeFormatter DATE_SHORT_FORMATTER =
            DateTimeFormatter.ofPattern("dd.MM.yyyy");

    private final CalendarDateService prodCalendarDateService;

    public Map<LocalDate, CalendarDateDto> loadByDates(Collection<LocalDate> requestedDates) {
        List<LocalDate> uniqueRequestedDates = requestedDates.stream()
                .distinct()
                .toList();

        List<String> dateShortKeys = uniqueRequestedDates.stream()
                .map(date -> date.format(DATE_SHORT_FORMATTER))
                .toList();

        List<CalendarDateDto> mdCalendarDateDtos =
                prodCalendarDateService.searchByDates(dateShortKeys);

        Map<LocalDate, CalendarDateDto> indexByDate = new HashMap<>(mdCalendarDateDtos.size() * 2);

        for (CalendarDateDto mdDto : mdCalendarDateDtos) {
            String dateShortFromMd = mdDto.getDateShort();

            if (dateShortFromMd == null) {
                throw new MdContractViolationException("MD returned dto with null dateShort");
            }

            LocalDate parsedDate = LocalDate.parse(dateShortFromMd, DATE_SHORT_FORMATTER);

            // если внезапно дубль — сохраняем первый
            indexByDate.putIfAbsent(parsedDate, mdDto);
        }

        return indexByDate;
    }
}
```
