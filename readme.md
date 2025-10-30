```java

/**
 * Сервис для кэширования реквизитов поставщиков.
 * <p>
 * Обеспечивает кэширование результатов запросов к мастер-данным по {@code supplierId}
 * с использованием Spring Cache (реализация — Caffeine).
 * </p>
 *
 * <h3>Кэш-регион</h3>
 * <ul>
 *   <li>{@code supplier_req_by_supplier_id} — хранит список {@link BankDto} по ключу {@code supplierId}.</li>
 * </ul>
 *
 * <p>Используется в {@link SupplierService2} для ускорения повторных запросов реквизитов.
 * Аннотация {@code @Cacheable(sync = true)} гарантирует, что при конкурентных запросах
 * к одному и тому же {@code supplierId} данные будут загружены из источника только один раз.</p>
 *
 * @author …
 * @see org.springframework.cache.annotation.Cacheable
 * @see SupplierService2
 * @since 1.0
 */

/**
   * Загружает и кэширует реквизиты поставщика по его идентификатору.
   * <p>
   * При первом обращении выполняет запрос к мастер-данным
   * и сохраняет результат в кэше региона {@code supplier_req_by_supplier_id}.
   * Повторные обращения с тем же {@code supplierId} возвращают данные из кэша.
   * </p>
   *
   * <h3>Кэширование:</h3>
   * <ul>
   *   <li>Регион: {@code supplier_req_by_supplier_id}</li>
   *   <li>Ключ: значение параметра {@code supplierId}</li>
   *   <li>Тип: синхронное кэширование ({@code sync = true})</li>
   * </ul>
   *
   * @param supplierId уникальный идентификатор поставщика
   * @return список {@link BankDto} с реквизитами поставщика
   * @throws org.springframework.cache.Cache.ValueRetrievalException при ошибке получения значения из кэша
   * @see BaseMasterDataRequestService#requestDataWithRefItemSlug(String, String, java.util.List)
   * @since 1.0
   */
```
