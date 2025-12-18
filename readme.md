```java

/**
 * DTO с информацией о регионе.
 * <p>
 * Используется в REST-ответах региона (поиск по коду/кодам и получение полного списка),
 * а также как элемент, который кэшируется и возвращается из сервисного слоя.
 *
 * @author
 * @since 1.0
 */

/**
 * Маппер данных региона из ответа Master Data (плоская Map со строковыми значениями)
 * в доменный DTO {@link RegionDto}.
 * <p>
 * Ожидаемые ключи в {@code item} соответствуют полям Master Data:
 * <ul>
 *   <li>{@code id}  → {@link RegionDto#uuid}</li>
 *   <li>{@code slug} → {@link RegionDto#regionCode}</li>
 *   <li>{@code name} → {@link RegionDto#regionName}</li>
 *   <li>{@code description} → {@link RegionDto#regionDescription}</li>
 * </ul>
 *
 * @author
 * @since 1.0
 */

  /**
     * Преобразует один элемент Master Data в {@link RegionDto}.
     *
     * @param item плоская структура (Map) с полями региона из Master Data
     * @return заполненный {@link RegionDto} или {@code null}, если {@code item} равен {@code null} либо пустой
     */
```
