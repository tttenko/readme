```java
/**
 * Политика выбора размера "окна" дат для расчёта диапазона типа {@code WORK}.
 *
 * <p>Окно — это список последовательных дат, для которых батчем загружаются данные МД,
 * чтобы попытаться найти конечную дату, на которой набирается нужное количество рабочих дней.</p>
 *
 * <p>Политика задаёт:
 * <ul>
 *   <li>начальный размер окна, исходя из {@code numDays};</li>
 *   <li>правило роста окна, если в текущем окне рабочих дней не хватило;</li>
 *   <li>верхнюю границу (защита от бесконечного роста и слишком больших запросов).</li>
 * </ul>
 */
@Component
public class WorkWindowPolicy {

    /** Минимальный размер окна (защита для маленьких numDays). */
    private static final int MIN_WINDOW = 30;

    /** Максимальный размер окна (защита от слишком больших запросов в МД). */
    private static final int MAX_WINDOW = 4000;

    /**
     * Вычисляет стартовый размер окна.
     *
     * <p>Логика: берём {@code numDays * 2 + 20} как эвристику, затем ограничиваем снизу/сверху.</p>
     *
     * @param numDays сколько рабочих дней нужно набрать
     * @return размер окна в диапазоне [{@link #MIN_WINDOW}, {@link #MAX_WINDOW}]
     */
    public int initialWindowSize(int numDays) {
        return Math.min(Math.max(numDays * 2 + 20, MIN_WINDOW), MAX_WINDOW);
    }

    /**
     * Вычисляет следующий размер окна, если в текущем окне не удалось набрать нужное число рабочих дней.
     *
     * <p>Логика роста:
     * <ul>
     *   <li>экспоненциальный рост: {@code current * 2};</li>
     *   <li>линейный рост: {@code current + max(50, numDays)} (чтобы гарантировать прогресс даже на малых значениях);</li>
     *   <li>выбираем максимум из двух вариантов и ограничиваем сверху {@link #MAX_WINDOW}.</li>
     * </ul>
     *
     * @param current текущий размер окна
     * @param numDays сколько рабочих дней нужно набрать (влияет на линейный шаг)
     * @return следующий размер окна (не больше {@link #MAX_WINDOW})
     */
    public int nextWindowSize(int current, int numDays) {
        int doubled = current * 2;
        int linear = current + Math.max(50, numDays);
        return Math.min(Math.max(doubled, linear), MAX_WINDOW);
    }

    /**
     * @return максимально допустимый размер окна
     */
    public int maxWindowSize() {
        return MAX_WINDOW;
    }
}
```
