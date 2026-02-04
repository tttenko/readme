```java

/**
 * Классификация диапазона дней, которую клиент передаёт в запросе.
 *
 * <p>Значение приходит как строка (wire-format) и может быть пустым/неизвестным.
 * В этом случае применяется безопасный дефолт {@link #ALL}.</p>
 */
public enum DaysClassification {

    /** Вернуть все дни подряд (календарный диапазон). */
    ALL("all"),

    /** Вернуть диапазон, в котором набирается N рабочих дней. */
    WORK("work");

    private final String wireValue;

    DaysClassification(String wireValue) {
        this.wireValue = wireValue;
    }

    /**
     * Парсит значение из запроса. Если значение отсутствует/пустое/не распознано —
     * возвращает {@link #ALL}.
     *
     * @param rawValue сырое значение параметра из запроса
     * @return распознанная классификация или {@link #ALL} по умолчанию
     */
    public static DaysClassification parseOrDefault(String rawValue) {
        if (rawValue == null || rawValue.isBlank()) {
            return ALL;
        }

        String normalizedValue = rawValue.trim().toLowerCase();

        for (DaysClassification classification : values()) {
            if (classification.wireValue.equals(normalizedValue)) {
                return classification;
            }
        }

        // неизвестное значение → безопасный дефолт
        return ALL;
    }
}
```
