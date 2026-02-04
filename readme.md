```java

@Component
@RequiredArgsConstructor
public class MdCalendarDataProvider implements CalendarDataProvider {

    private static final DateTimeFormatter DDMMYYYY = DateTimeFormatter.ofPattern("dd.MM.yyyy");
    private final CalendarDateService prodCalendarDateService;

    @Override
    public Map<LocalDate, CalendarDateDto> loadByDates(Collection<LocalDate> dates) {
        // сохраняем порядок и убираем дубли
        List<LocalDate> unique = dates.stream().distinct().toList();
        List<String> keys = unique.stream().map(d -> d.format(DDMMYYYY)).toList();

        List<CalendarDateDto> dtos = prodCalendarDateService.searchByDates(keys);

        // Индексируем по LocalDate из dto.getDateShort()
        Map<LocalDate, CalendarDateDto> index = new HashMap<>(dtos.size() * 2);
        for (CalendarDateDto dto : dtos) {
            String shortStr = dto.getDateShort();
            if (shortStr == null) {
                throw new IllegalStateException("MD returned dto with null dateShort");
            }
            LocalDate ld = LocalDate.parse(shortStr, DDMMYYYY);
            index.putIfAbsent(ld, dto); // или бросать на дубль
        }
        return index;
    }
}


```
