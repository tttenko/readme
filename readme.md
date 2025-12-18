```java
/**
 * Сервис для работы со справочником регионов (Master Data).
 * <p>
 * Предоставляет:
 * <ul>
 *   <li>поиск регионов по коду(ам) через {@link CacheGetOrLoadService} (batch-cache);</li>
 *   <li>получение полного списка регионов через {@link RegionCacheOps} (отдельный кеш /all).</li>
 * </ul>
 *
 * @author
 * @since 1.0
 */

 /**
   * Возвращает список регионов по массиву кодов регионов.
   * <p>
   * Использует {@link CacheGetOrLoadService} для получения данных из кеша или загрузки отсутствующих
   * значений через соответствующий loader (например, {@code LoaderRegionByCode}).
   *
   * @param regionCodes список кодов регионов; не должен быть {@code null}
   * @return список найденных регионов (может быть пустым)
   * @throws NullPointerException если {@code regionCodes == null}
   * @see #REGION_BY_CODE
   */

   /**
   * Возвращает полный перечень регионов.
   * <p>
   * Данные берутся из кеша, наполняемого через {@link RegionCacheOps#loadAllRegions()}.
   * Обычно используется ручкой вида {@code GET /api/v1/info/region_code/all}.
   *
   * @return полный список регионов (может быть пустым)
   * @see RegionCacheOps#loadAllRegions()
   */
```
