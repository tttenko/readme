```java
/**
 * Калькулятор диапазона типа {@code WORK}: подбирает конечную дату так, чтобы в непрерывном
 * промежутке от {@code start} набралось {@code numDays} рабочих дней.
 *
 * <p>Так как конечная дата заранее неизвестна, используется "оконный" алгоритм:
 * <ol>
 *   <li>Берём начальный размер окна из {@link WorkWindowPolicy}.</li>
 *   <li>Генерируем окно дат в направлении {@code step} (порядок важен).</li>
 *   <li>Догружаем данные из МД только для отсутствующих дат и складываем их в локальный индекс.</li>
 *   <li>Пробегаем окно по порядку и пытаемся найти дату, на которой накопилось {@code numDays} рабочих дней.</li>
 *   <li>Если не нашли — увеличиваем окно по политике и повторяем.</li>
 * </ol>
 *
 * <p>Ошибки/инварианты:
 * <ul>
 *   <li>Если политика роста окна не увеличивает размер — это ошибка конфигурации ({@link IllegalStateException}).</li>
 *   <li>Если нужный конец так и не найден до {@code maxWindowSize} — считается нарушением инварианта
 *       (слишком маленький maxWindow, нет данных МД в нужной области или аномальные входные параметры).</li>
 *   <li>Если в МД нет данных для даты, необходимой для подсчёта — выбрасывается {@link MissingCalendarDataException}.</li>
 * </ul>
 */
@Component
@RequiredArgsConstructor
public class TypeWorkRangeCalculator implements RangeCalculator {

    private final WorkWindowPolicy windowPolicy;
    private final CalendarDataProvider dataProvider;
    private final CalendarRangeHelpers calendarRangeHelpers;

    /**
     * Вычисляет диапазон дат, в котором набирается {@code numDays} рабочих дней, начиная от {@code start}.
     *
     * @param start   стартовая дата
     * @param numDays требуемое количество рабочих дней (>= 1)
     * @param step    направление: {@code +1} — вперёд, {@code -1} — назад
     * @return непрерывный диапазон дат (всегда по возрастанию, через {@code inclusiveRange})
     */
    @Override
    public List<LocalDate> calculate(LocalDate start, int numDays, int step) {
        int windowSize = windowPolicy.initialWindowSize(numDays);

        // локальный кэш в рамках одного расчёта: чтобы не перезагружать одни и те же даты при росте окна
        Map<LocalDate, CalendarDateDto> mdIndex = new HashMap<>();

        while (windowSize <= windowPolicy.maxWindowSize()) {
            List<LocalDate> windowDates = calendarRangeHelpers.generateWindow(start, windowSize, step);

            List<LocalDate> missingDates = windowDates.stream()
                    .filter(date -> !mdIndex.containsKey(date))
                    .toList();

            if (!missingDates.isEmpty()) {
                mdIndex.putAll(dataProvider.loadByDates(missingDates));
            }

            Optional<LocalDate> endDateOpt = findEndDate(windowDates, mdIndex, numDays);
            if (endDateOpt.isPresent()) {
                return calendarRangeHelpers.inclusiveRange(start, endDateOpt.get());
            }

            int nextWindowSize = windowPolicy.nextWindowSize(windowSize, numDays);
            if (nextWindowSize <= windowSize) {
                throw new IllegalStateException(
                        "WorkWindowPolicy must increase window size: current=" + windowSize + ", next=" + nextWindowSize
                );
            }

            windowSize = nextWindowSize;
        }

        throw new IllegalStateException(
                "Invariant violated: work-range end not found. start=" + start +
                ", numDays=" + numDays +
                ", step=" + step +
                ", maxWindow=" + windowPolicy.maxWindowSize()
        );
    }

    /**
     * Ищет конечную дату внутри окна: проходит даты по порядку окна и считает рабочие дни,
     * пока не наберётся {@code targetWorkDays}.
     *
     * @param orderedDates    окно дат в порядке направления ({@code step})
     * @param mdIndex         индекс данных МД по датам
     * @param targetWorkDays  сколько рабочих дней нужно набрать
     * @return дата, на которой накопилось нужное количество рабочих дней, либо {@link Optional#empty()}
     * @throws MissingCalendarDataException если для какой-то даты из окна отсутствуют данные МД
     */
    private Optional<LocalDate> findEndDate(
            List<LocalDate> orderedDates,
            Map<LocalDate, CalendarDateDto> mdIndex,
            int targetWorkDays
    ) {
        if (targetWorkDays <= 0) {
            return Optional.empty();
        }

        int workdaysCount = 0;

        for (LocalDate date : orderedDates) {
            CalendarDateDto mdDto = requireDto(date, mdIndex);

            if (calendarRangeHelpers.isWorkday(mdDto.getDateType())) {
                workdaysCount++;
                if (workdaysCount == targetWorkDays) {
                    return Optional.of(date);
                }
            }
        }

        return Optional.empty();
    }

    /**
     * Достаёт DTO по дате из индекса. Если данных нет — это критично для подсчёта,
     * поэтому выбрасывается доменное исключение.
     *
     * @param date    дата, для которой нужны данные МД
     * @param mdIndex индекс данных МД
     * @return DTO из МД
     * @throws MissingCalendarDataException если данных для даты нет
     */
    private CalendarDateDto requireDto(LocalDate date, Map<LocalDate, CalendarDateDto> mdIndex) {
        CalendarDateDto dto = mdIndex.get(date);
        if (dto == null) {
            throw new MissingCalendarDataException(date);
        }
        return dto;
    }
}
```
