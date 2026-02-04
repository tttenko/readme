```java

/**
 * Провайдер данных производственного календаря из МД: батч-загрузка по списку дат
 * и построение индекса {@code Map<LocalDate, CalendarDateDto>} для быстрого доступа.
 */

 /**
     * Загружает календарные DTO из МД одним запросом и индексирует их по {@link LocalDate}.
     *
     * @param requestedDates даты, по которым нужно получить данные (дубликаты допускаются)
     * @return индекс DTO по дате
     * @throws MdContractViolationException если МД вернул некорректные данные (например, {@code dateShort == null}
     *                                     или не парсится форматом {@code dd.MM.yyyy})
     */

     /**
 * Вспомогательные методы для расчёта диапазонов дат.
 *
 * <p>Используется в алгоритме построения диапазона:
 * <ul>
 *   <li>для "оконной" загрузки — генерирует список дат фиксированной длины в нужном направлении;</li>
 *   <li>для финального ответа — строит непрерывный диапазон между двумя датами включительно
 *       (всегда по возрастанию, независимо от направления запроса).</li>
 * </ul>
 */
@Component
public class CalendarRangeHelpers {

    /**
     * Генерирует "окно" дат фиксированного размера в заданном направлении.
     *
     * <p>Пример: start=01.01, windowSize=3, step=+1 → [01.01, 02.01, 03.01].</p>
     *
     * @param startDate   дата начала окна
     * @param windowSize  количество дат в окне (должно быть >= 0)
     * @param dayStep     шаг по дням: {@code +1} вперёд или {@code -1} назад
     * @return список дат в порядке направления шага
     */
    public List<LocalDate> generateWindow(LocalDate startDate, int windowSize, int dayStep) {
        List<LocalDate> windowDates = new ArrayList<>(windowSize);
        LocalDate currentDate = startDate;

        for (int i = 0; i < windowSize; i++) {
            windowDates.add(currentDate);
            currentDate = currentDate.plusDays(dayStep);
        }

        return windowDates;
    }

    /**
     * Строит непрерывный диапазон дат между двумя датами включительно.
     *
     * <p>Диапазон всегда возвращается по возрастанию (это важно для ответа API),
     * даже если входные даты переданы в обратном порядке.</p>
     *
     * @param firstDate  первая дата (граница диапазона)
     * @param secondDate вторая дата (граница диапазона)
     * @return список дат от меньшей к большей, включая обе границы
     */
    public List<LocalDate> inclusiveRange(LocalDate firstDate, LocalDate secondDate) {
        LocalDate rangeStart = firstDate.isBefore(secondDate) ? firstDate : secondDate;
        LocalDate rangeEnd = firstDate.isBefore(secondDate) ? secondDate : firstDate;

        List<LocalDate> rangeDates = new ArrayList<>();
        LocalDate currentDate = rangeStart;

        while (!currentDate.isAfter(rangeEnd)) {
            rangeDates.add(currentDate);
            currentDate = currentDate.plusDays(1);
        }

        return rangeDates;
    }

    /**
     * Определяет, является ли день рабочим по значению {@code dateType} из МД.
     *
     * @param dateType тип дня из МД
     * @return {@code true}, если тип соответствует рабочему дню
     */
    public boolean isWorkday(String dateType) {
        return "1".equals(dateType) || "2".equals(dateType);
    }
}
```
