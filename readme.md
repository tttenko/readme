```java
/**
 * Маппер ответа диапазона: преобразует список дат и данные МД в список {@link CalendarRangeItemDto}.
 *
 * <p>Ожидается, что {@code ascendingDates} уже отсортирован по возрастанию.
 * Для каждой даты берётся DTO из {@code md} и собирается элемент ответа (включая признак "рабочий/нерабочий").</p>
 *
 * @throws MissingCalendarDataException если для какой-либо даты нет данных в {@code md}
 */

 /**
 * Собирает список элементов ответа {@link CalendarRangeItemDto} для диапазона дат.
 *
 * <p>Входные {@code ascendingDates} должны быть отсортированы по возрастанию.
 * Для каждой даты метод берёт соответствующий {@link CalendarDateDto} из {@code mdIndex}
 * и формирует элемент ответа (включая статус "рабочий/нерабочий").</p>
 *
 * @param ascendingDates даты диапазона в порядке возрастания
 * @param mdIndex индекс данных МД по дате
 * @return элементы ответа в том же порядке, что и {@code ascendingDates}
 * @throws MissingCalendarDataException если для какой-либо даты нет записи в {@code mdIndex}
 */
```
