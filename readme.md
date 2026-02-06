```java
/**
 * Политика выбора размера «окна» дат для расчёта диапазона типа WORK.
 * Стартовый размер: {@code numDays * 2 + BUFFER}, затем рост шагом {@code GROW_STEP}
 * с ограничением {@code [MIN_WINDOW..MAX_WINDOW]}.
 */
@Component
public class WorkdayRangeWindowSizePolicy {

    private static final int MIN_WINDOW = 30;
    private static final int BUFFER = 30;      // запас на праздники/выходные
    private static final int GROW_STEP = 30;   // шаг увеличения окна
    private static final int MAX_WINDOW = 366; // календарь на год (високосный)

    /** Возвращает стартовый размер окна для указанного числа рабочих дней. */
    public int initialWindowSize(int numDays) {
        int base = numDays * 2 + BUFFER;
        return clamp(base, MIN_WINDOW, MAX_WINDOW);
    }

    /** Возвращает следующий размер окна, если в текущем не удалось найти конец диапазона. */
    public int nextWindowSize(int currentWindowSize) {
        return clamp(currentWindowSize + GROW_STEP, MIN_WINDOW, MAX_WINDOW);
    }

    /** Максимально допустимый размер окна. */
    public int maxWindowSize() {
        return MAX_WINDOW;
    }

    private int clamp(int value, int min, int max) {
        return Math.min(Math.max(value, min), max);
    }
}
```
