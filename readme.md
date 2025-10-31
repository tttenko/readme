```java
/**
 * Unit-тесты для {@link SupplierCacheOps}.
 * <p>
 * Проверяют работу аннотации {@link org.springframework.cache.annotation.Cacheable}
 * при загрузке реквизитов поставщика:
 * корректное кэширование по ключу {@code supplierId}, отсутствие повторных обращений
 * к {@link BaseMasterDataRequestService} и независимость записей для разных идентификаторов.
 */
@ExtendWith(SpringExtension.class)
class SupplierCacheOpsTest {

    /**
     * Проверяет сценарий кэш-хита для одного {@code supplierId}.
     * <p>
     * GIVEN: кэш пуст<br>
     * WHEN: метод {@link SupplierCacheOps#loadBySupplierId(String)} вызывается дважды<br>
     * THEN: запрос к {@link BaseMasterDataRequestService} выполняется один раз,
     * а результат сохраняется и повторно возвращается из кэша.
     */
    @Test
    void givenId_whenCalledTwice_thenBackendOnce_andCachedUnderIdKey() { ... }

    /**
     * Проверяет, что для разных {@code supplierId} создаются независимые записи в кэше.
     * <p>
     * GIVEN: кэш пуст<br>
     * WHEN: вызывается {@link SupplierCacheOps#loadBySupplierId(String)} для двух разных ID<br>
     * THEN: каждый ID запрашивается в {@link BaseMasterDataRequestService} ровно один раз,
     * а кэш содержит по одному элементу на каждый ключ.
     */
    @Test
    void givenTwoDifferentIds_whenCall_thenTwoLoads_andTwoCacheEntries() { ... }
}


```
