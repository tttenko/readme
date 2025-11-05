```java

/**
 * Набор удобных ассертов для проверки содержимого Spring-кэша в тестах.
 * <p>Работает поверх {@code CacheIntrospection} и не зависит от конкретной реализации кэша
 * (Caffeine/ConcurrentMap и т.д.).</p>
 *
 * <h4>Что умеет</h4>
 * <ul>
 *   <li>Проверка набора ключей в кэше;</li>
 *   <li>Проверка пустоты кэша;</li>
 *   <li>Получение типизированного «бакета» по ключу;</li>
 *   <li>Проверки размера и содержимого бакета.</li>
 * </ul>
 *
 * @author …
 * @since 1.0
 */
public final class CacheAssertions {
java
Копировать код
/**
 * Проверяет, что в кэше присутствуют ровно указанные ключи
 * (порядок не важен, дубликаты не допускаются).
 *
 * @param mgr         {@link CacheManager}, из которого получаем кэш
 * @param cacheName   имя кэша
 * @param expectedKeys ожидаемые ключи
 * @throws AssertionError если набор ключей не совпал
 */
public static void assertCacheHasKeys(CacheManager mgr, String cacheName, String... expectedKeys) { ... }
java
Копировать код
/**
 * Проверяет, что кэш пуст.
 *
 * @param mgr       {@link CacheManager}
 * @param cacheName имя кэша
 * @throws AssertionError если кэш содержит хотя бы один ключ
 */
public static void assertCacheEmpty(CacheManager mgr, String cacheName) { ... }
java
Копировать код
/**
 * Возвращает типизированный бакет по ключу, предварительно убеждаясь,
 * что под ключом лежит объект нужного типа.
 *
 * @param mgr       {@link CacheManager}
 * @param cacheName имя кэша
 * @param key       ключ, под которым хранится бакет
 * @return найденный {@code RateBucket}
 * @throws AssertionError если по ключу нет значения или тип отличается
 */
public static RateBucket bucket(CacheManager mgr, String cacheName, String key) { ... }
java
Копировать код
/**
 * Проверяет ожидаемый размер набора элементов внутри бакета.
 *
 * @param mgr       {@link CacheManager}
 * @param cacheName имя кэша
 * @param key       ключ бакета
 * @param expected  ожидаемое количество элементов в бакете
 * @throws AssertionError если размер не совпал
 */
public static void assertBucketSize(CacheManager mgr, String cacheName, String key, int expected) { ... }
java
Копировать код
/**
 * Проверяет, что бакет содержит элементы с указанными идентификаторами.
 * Сравнение идёт по {@code NdsFullDto#getId()} и не учитывает порядок.
 *
 * @param mgr       {@link CacheManager}
 * @param cacheName имя кэша
 * @param key       ключ бакета
 * @param ids       ожидаемые идентификаторы элементов
 * @throws AssertionError если хотя бы один id не найден
 */
public static void assertBucketContainsIds(CacheManager mgr, String cacheName, String key, String... ids) { ... }
java
Копировать код
// приватный конструктор — без JavaDoc
private CacheAssertions() {}

```
