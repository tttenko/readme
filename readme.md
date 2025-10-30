```java

Класс SupplierService2
/**
 * Сервис для получения и кэширования данных о поставщиках и контрагентах.
 * <p>Использует {@link BatchCacheSupport} для батч-загрузки промахов и частичного кэширования.
 * Кэш реализован через Spring Cache (Caffeine) и управляется по регионам:</p>
 *
 * <ul>
 *   <li>{@code supplier_req_by_supplier_id} — кэш реквизитов поставщика по {@code supplierId};</li>
 *   <li>{@code supplier_by_id} — кэш контрагентов по {@code id};</li>
 *   <li>{@code supplier_by_inn_kpp} — кэш контрагентов по комбинации {@code inn+kpp}.</li>
 * </ul>
 *
 * <p>Если передан список ключей, сервис подгружает из мастер-данных только отсутствующие значения
 * и заполняет кэш пер-ключево, без дублирования «агрегатных» записей.</p>
 *
 * @implNote Батч-загрузки выполняются потокобезопасно и не требуют внешней синхронизации.
 * @since 1.0
 */

🔹 Метод searchSupplierRequisite
/**
 * Возвращает реквизиты поставщика по списку идентификаторов.
 * <p>Использует регион кэша {@code supplier_req_by_supplier_id}, где ключом выступает {@code supplierId}.</p>
 *
 * @param ids список идентификаторов поставщиков (null/blank значения игнорируются)
 * @return результат со списком {@link BankDto}, соответствующих указанным {@code supplierId}
 * @see BatchCacheSupport#fetchBatch(String, java.util.List, java.util.function.Function, java.util.function.Function, Class)
 * @since 1.0
 */

🔹 Метод searchCounterpartiesByCriteria
/**
 * Возвращает список контрагентов по критериям (ИНН и/или КПП).
 * <p>Если заданы оба параметра, выполняется поиск с кэшированием
 * в регионе {@code supplier_by_inn_kpp}, где ключ = {@code inn:%s:kpp:%s}.
 * Если задан только один параметр (ИНН или КПП), выполняется прямой запрос без кэша.</p>
 *
 * @param inn ИНН контрагента (опционально)
 * @param kpp КПП контрагента (опционально)
 * @return успешный результат со списком {@link CounterpartyDto}
 * @see #buildCriteriaMap(String, String)
 * @see #buildInnKppKey(String, String)
 * @since 1.0
 */

🔹 Метод getCounterpartiesById
/**
 * Возвращает контрагентов по списку идентификаторов.
 * <p>Использует регион кэша {@code supplier_by_id}, где ключом является {@code id}.</p>
 *
 * @param ids список идентификаторов контрагентов
 * @return успешный результат со списком {@link CounterpartyDto}
 * @see BatchCacheSupport#fetchBatch(String, java.util.List, java.util.function.Function, java.util.function.Function, Class)
 * @since 1.0
 */

🔹 Метод loadRequisitesBySupplierIds
/**
 * Батч-загрузчик реквизитов поставщиков по идентификаторам из мастер-данных.
 * <p>Вызывается при промахах в регионе {@code supplier_req_by_supplier_id}.</p>
 *
 * @param ids список идентификаторов поставщиков
 * @return список {@link BankDto} с реквизитами
 * @see BaseMasterDataRequestService#requestDataWithRefItemSlug2(String, String, java.util.List)
 * @since 1.0
 */

🔹 Метод loadSuppliersByIds
/**
 * Батч-загрузчик контрагентов из мастер-данных по списку идентификаторов.
 * <p>Вызывается при промахах в регионе {@code supplier_by_id}.</p>
 *
 * @param ids список идентификаторов контрагентов
 * @return список {@link CounterpartyDto}
 * @see BaseMasterDataRequestService#requestDataWithAttribute(String, java.util.List, SearchRequestProperties.Context)
 * @since 1.0
 */

🔹 Метод loadSuppliersByCriteria
/**
 * Загружает контрагентов из мастер-данных по критериям (ИНН/КПП).
 * <p>Используется как источник данных при промахах в регионе {@code supplier_by_inn_kpp}.</p>
 *
 * @param criteria карта фильтров (ИНН и КПП) для запроса к мастер-данным
 * @return список {@link CounterpartyDto}
 * @see BaseMasterDataRequestService#requestDataWithAttribute(String, java.util.Map)
 * @since 1.0
 */

🔹 Метод getSupplierByCriteria
/**
 * Выполняет прямой поиск контрагентов по критериям без кэширования.
 * <p>Используется, если указан только один параметр (ИНН или КПП).</p>
 *
 * @param criteria карта критериев (ИНН и/или КПП)
 * @return результат со списком {@link CounterpartyDto}
 * @since 1.0
 */

🔹 Метод buildCriteriaMap
/**
 * Формирует карту критериев для поиска контрагентов по ИНН и/или КПП.
 * <p>Null или пустые значения не включаются в результат.</p>
 *
 * @param inn значение ИНН (может быть {@code null} или blank)
 * @param kpp значение КПП (может быть {@code null} или blank)
 * @return карта критериев, содержащая 0–2 записи
 * @since 1.0
 */

🔹 Метод buildInnKppKey
/**
 * Формирует детерминированный ключ кэша по паре ИНН и КПП.
 *
 * @param inn значение ИНН
 * @param kpp значение КПП
 * @return строка-ключ формата {@code inn:%s:kpp:%s}
 * @since 1.0
 */
```
