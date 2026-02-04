```java
/**
 * Возвращает 404, если для одной или нескольких дат нет записей календаря в МД.
 * Используется для случаев, когда без этих данных невозможно корректно собрать ответ.
 */

 /**
 * Возвращает 502, если МД нарушило контракт ответа (например, вернуло DTO с некорректными/пустыми полями).
 * Считаем это ошибкой внешнего сервиса, а не пользователя.
 */

 /**
 * Ошибка "нарушение контракта" со стороны сервиса МД.
 * <p>
 * Бросается, когда МД вернул ответ, который не соответствует ожидаемой схеме/инвариантам
 * (например, обязательное поле отсутствует или имеет некорректный формат).
 * Обычно маппится в HTTP 502 (Bad Gateway) на уровне {@code GlobalExceptionHandler}.
 */

  /**
     * @param message человекочитаемое описание того, какой именно инвариант/контракт нарушен
     */
/**
 * Отсутствуют данные календаря из МД для конкретной даты.
 * <p>
 * Используется как доменное исключение, когда алгоритм не может продолжить расчёт диапазона
 * или сформировать ответ, потому что МД не вернул запись для одной из нужных дат.
 * Обычно маппится в HTTP 404 (Not Found) на уровне {@code GlobalExceptionHandler}.
 */
@Getter
public class MissingCalendarDataException extends RuntimeException {

    private static final DateTimeFormatter DDMMYYYY = DateTimeFormatter.ofPattern("dd.MM.yyyy");

    /**
     * Дата, для которой отсутствуют данные в МД.
     */
    private final LocalDate date;

    /**
     * @param date дата, для которой не удалось получить календарные данные из МД
     */
    public MissingCalendarDataException(LocalDate date) {
        super("No calendar data from MD for date: " + format(date));
        this.date = date;
    }

    /**
     * Короткое строковое представление даты в формате {@code dd.MM.yyyy}.
     * <p>
     * Удобно для подстановки в локализованное сообщение об ошибке (например, как аргумент i18n-шаблона).
     *
     * @return дата в формате {@code dd.MM.yyyy}
     */
    public String getDateShort() {
        return format(date);
    }

    /**
     * Форматирует дату в {@code dd.MM.yyyy}.
     * Используется для сообщений/логов, чтобы не размазывать форматирование по коду.
     */
    private static String format(LocalDate date) {
        return date == null ? "null" : date.format(DDMMYYYY);
    }
}

 
```
