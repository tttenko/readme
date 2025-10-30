```java

/**
 * Unit-тесты для {@link SupplierService2}.
 * <p>
 * Проверяют бизнес-логику и оркестрацию сервисного слоя:
 * корректность выбора стратегии загрузки данных, интеграцию с {@link BatchCacheSupport},
 * взаимодействие с {@link SupplierCacheOps} и работу с {@link org.springframework.cache.CacheManager}.
 * <p>
 * Цель тестов — убедиться, что сервис правильно использует кэширование и не обращается
 * к мастер-данным (MDM) при наличии валидных данных в кэше.
 *
 * @author Твой_ник
 */
@ExtendWith(MockitoExtension.class)
class SupplierService2Test {

    /**
     * Проверяет поведение метода {@link SupplierService2#searchCounterpartiesByCriteria(String, String)}
     * при наличии обоих параметров — INN и KPP.
     * <p>
     * Ожидается, что сервис:
     * <ul>
     *   <li>использует {@link BatchCacheSupport#fetchBatch} для получения данных из кэша;</li>
     *   <li>не вызывает фолбэк-загрузку из мастер-данных;</li>
     *   <li>формирует корректный ключ кэша вида {@code "inn:<inn>:kpp:<kpp>"}.</li>
     * </ul>
     */
    @Test
    void searchCounterpartiesByCriteria_innAndKpp_hitsBatch_noFallback() { ... }

    /**
     * Проверяет поведение метода {@link SupplierService2#searchCounterpartiesByCriteria(String, String)}
     * при задании только INN без KPP.
     * <p>
     * Ожидается, что:
     * <ul>
     *   <li>батч-кэш не используется;</li>
     *   <li>сервис вызывает прямой запрос к мастер-данным через {@link BaseMasterDataRequestService};</li>
     *   <li>кэш {@link SupplierService2#SUPPLIER_BY_INN_KPP} остаётся пустым.</li>
     * </ul>
     */
    @Test
    void searchCounterpartiesByCriteria_innOnly_bypassBatch_callsDirect() { ... }

    /**
     * Проверяет сценарий промаха батч-кэша в методе
     * {@link SupplierService2#searchCounterpartiesByCriteria(String, String)} при поиске по INN+KPP.
     * <p>
     * Ожидается, что:
     * <ul>
     *   <li>сначала происходит неудачная попытка загрузки из кэша (промах);</li>
     *   <li>далее выполняется загрузка из мастер-данных через {@link BaseMasterDataRequestService};</li>
     *   <li>результат сохраняется в кэш вручную через {@link CacheManager}.</li>
     * </ul>
     */
    @Test
    void searchCounterpartiesByCriteria_innAndKpp_batchMiss_fallbackAndCachePut() { ... }

    /**
     * Проверяет метод {@link SupplierService2#getCounterpartiesById(List)}.
     * <p>
     * Ожидается, что:
     * <ul>
     *   <li>для указанного списка идентификаторов вызывается {@link BatchCacheSupport#fetchBatch};</li>
     *   <li>прямая загрузка из мастер-данных не выполняется;</li>
     *   <li>результат возвращается в виде {@code ResultObj<List<CounterpartyDto>>}.</li>
     * </ul>
     */
    @Test
    void getCounterpartiesById_usesBatchLoad() { ... }

    /**
     * Проверяет метод {@link SupplierService2#searchSupplierRequisite(List)}.
     * <p>
     * Ожидается, что:
     * <ul>
     *   <li>сервис корректно фильтрует {@code null}, пустые и пробельные идентификаторы поставщиков;</li>
     *   <li>батч-кэш не используется;</li>
     *   <li>для валидных идентификаторов вызывается {@link SupplierCacheOps#loadBySupplierId(String)};</li>
     *   <li>все результаты объединяются и возвращаются в виде {@code ResultObj<List<BankDto>>}.</li>
     * </ul>
     */
    @Test
    void searchSupplierRequisite_filtersBlanks_usesSupplierCacheOps() { ... }
}


```
