```java
/**
 * Кеш-операции для получения полного перечня регионов из Master Data.
 * <p>
 * Используется для отдельной "явной" ручки вида {@code /all} (без передачи кодов в query),
 * чтобы не допускать поведения "если параметров нет — вернуть всё" в поисковой ручке.
 * <p>
 * Результат сохраняется в кеш {@link #REGION_ALL} по ключу {@code "ALL"} (с {@code sync=true}),
 * чтобы при параллельных запросах данные загружались из Master Data только один раз.
 *
 * @see BaseMasterDataRequestService
 * @see SearchRequestProperties
 * @see RegionMapper
 */

 /**
   * Загружает полный перечень регионов из Master Data и возвращает DTO-список.
   * <p>
   * Вызывает {@link BaseMasterDataRequestService#requestData(String, List, SearchRequestProperties.Context)}
   * с {@code searchFieldValue = null}, что трактуется как "получить всё" для заданного справочника.
   * <p>
   * Результат маппится через {@link RegionMapper#mapItemToDto(java.util.Map)} и кешируется.
   *
   * @return список регионов (может быть пустым)
   * @throws ru.sber.cs.supplier.portal.masterdata.exception.MdaMdErrorResponseException
   *         если Master Data вернул неуспешный статус (semantic != {@code "S"})
   * @see #REGION_ALL
   */
```
