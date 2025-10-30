```java

/**
 * Unit-тесты для {@link SupplierService2}.
 * <p>
 * Проверяют корректную оркестрацию сервисного слоя: выбор стратегии загрузки данных,
 * использование {@link BatchCacheSupport}, взаимодействие с {@link SupplierCacheOps}
 * и работу с кэшем через {@link org.springframework.cache.CacheManager}.
 *
 * @author Твой_ник
 */
@ExtendWith(MockitoExtension.class)
class SupplierService2Test {

    /**
     * Проверяет, что при наличии и INN, и KPP
     * сервис использует {@link BatchCacheSupport} и не выполняет фолбэк в мастер-данные.
     *
     * @see SupplierService2#searchCounterpartiesByCriteria(String, String)
     */
    @Test
    void searchCounterpartiesByCriteria_innAndKpp_hitsBatch_noFallback() { ... }

    /**
     * Проверяет, что при задании только INN
     * сервис пропускает батч-кэш и вызывает прямую загрузку из мастер-данных.
     *
     * @see SupplierService2#searchCounterpartiesByCriteria(String, String)
     */
    @Test
    void searchCounterpartiesByCriteria_innOnly_bypassBatch_callsDirect() { ... }

    /**
     * Проверяет сценарий промаха батч-кэша (INN+KPP):
     * происходит загрузка из мастер-данных и ручная запись результата в кэш.
     *
     * @see SupplierService2#searchCounterpartiesByCriteria(String, String)
     */
    @Test
    void searchCounterpartiesByCriteria_innAndKpp_batchMiss_fallbackAndCachePut() { ... }

    /**
     * Проверяет, что при поиске по списку идентификаторов
     * используется батч-загрузка через {@link BatchCacheSupport}.
     *
     * @see SupplierService2#getCounterpartiesById(List)
     */
    @Test
    void getCounterpartiesById_usesBatchLoad() { ... }

    /**
     * Проверяет, что при поиске реквизитов по supplierId
     * сервис фильтрует пустые идентификаторы и вызывает {@link SupplierCacheOps}.
     *
     * @see SupplierService2#searchSupplierRequisite(List)
     */
    @Test
    void searchSupplierRequisite_filtersBlanks_usesSupplierCacheOps() { ... }
}

}

```
