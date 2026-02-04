```java

/**
 * Сервис построения диапазона календарных дней по входным параметрам запроса.
 *
 * <p>Алгоритм:
 * <ol>
 *   <li>Парсит тип диапазона ({@code all}/{@code work}) с дефолтом.</li>
 *   <li>Определяет направление расчёта (вперёд/назад) через {@code step = +1/-1}.</li>
 *   <li>Считает список дат выбранным калькулятором (ALL или WORK).</li>
 *   <li>Батчем загружает данные по полученным датам из МД.</li>
 *   <li>Маппит результат в DTO ответа.</li>
 * </ol>
 */
@Service
@RequiredArgsConstructor
public class CalendarRangeService {

    private final CalendarDataProvider dataProvider;
    private final TypeAllRangeCalculator allCalc;
    private final TypeWorkRangeCalculator workCalc;
    private final CalendarRangeMapper calendarRangeMapper;

    /**
     * Строит диапазон календарных дней от стартовой даты.
     *
     * @param start      стартовая дата диапазона
     * @param numDays    требуемое количество дней (>= 1)
     * @param isForward  направление расчёта: {@code true} — вперёд, {@code false} — назад
     * @param raw        тип диапазона: {@code all} или {@code work} (пустое/ null → {@code all})
     * @return список элементов диапазона (в ответе будет отсортирован по возрастанию на этапе маппинга/калькулятора)
     */
    public List<CalendarRangeItemDto> buildRange(
            LocalDate start,
            int numDays,
            boolean isForward,
            String raw
    ) {
        DaysClassification daysClassification = DaysClassification.parseOrDefault(raw);
        int step = isForward ? 1 : -1;

        List<LocalDate> dates = switch (daysClassification) {
            case ALL -> allCalc.calculate(start, numDays, step);
            case WORK -> workCalc.calculate(start, numDays, step);
        };

        Map<LocalDate, CalendarDateDto> md = dataProvider.loadByDates(dates);
        return calendarRangeMapper.map(dates, md);
    }
}
```
