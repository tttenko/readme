```java
/**
 * Контракт для вычисления списка дат диапазона по заданным параметрам.
 *
 * <p>Реализации могут строить:
 * <ul>
 *   <li>календарный диапазон из {@code numDays} подряд (ALL);</li>
 *   <li>диапазон, в котором набирается {@code numDays} рабочих дней (WORK).</li>
 * </ul>
 *
 * <p>Направление задаётся параметром {@code step}:
 * {@code +1} — вперёд по времени, {@code -1} — назад.</p>
 */
public interface RangeCalculator {

    /**
     * Вычисляет список дат диапазона.
     *
     * @param start  стартовая дата расчёта
     * @param numDays количество дней (или рабочих дней, в зависимости от реализации), которые нужно набрать (>= 1)
     * @param step   шаг по дням: {@code +1} или {@code -1}
     * @return список дат, полученный алгоритмом реализации
     */
    List<LocalDate> calculate(LocalDate start, int numDays, int step);
}

/**
 * Калькулятор диапазона типа {@code ALL}: возвращает {@code numDays} календарных дней подряд.
 *
 * <p>Алгоритм:
 * <ol>
 *   <li>Вычисляет конечную дату: {@code end = start + step * (numDays - 1)}.</li>
 *   <li>Возвращает непрерывный диапазон дат между {@code start} и {@code end} включительно
 *       (через {@link CalendarRangeHelpers#inclusiveRange(LocalDate, LocalDate)}).</li>
 * </ol>
 *
 * <p>Важно: даже при {@code step = -1} итоговый список будет отсортирован по возрастанию,
 * потому что {@code inclusiveRange} всегда возвращает даты в порядке от меньшей к большей.</p>
 */
@Component
@RequiredArgsConstructor
public class TypeAllRangeCalculator implements RangeCalculator {

    private final CalendarRangeHelpers calendarRangeHelpers;

    /**
     * Вычисляет календарный диапазон из {@code numDays} подряд идущих дней.
     *
     * @param start   стартовая дата
     * @param numDays количество дней (>= 1)
     * @param step    направление: {@code +1} — вперёд, {@code -1} — назад
     * @return список дат диапазона (всегда по возрастанию)
     */
    @Override
    public List<LocalDate> calculate(LocalDate start, int numDays, int step) {
        LocalDate end = start.plusDays((long) step * (numDays - 1));
        return calendarRangeHelpers.inclusiveRange(start, end);
    }
}


```
