```java

/**
 * Утилиты для инспекции содержимого Spring Cache во время тестов.
 * <p>
 * Работает с кешами типа {@link org.springframework.cache.concurrent.ConcurrentMapCache}
 * (менеджер, например, {@link org.springframework.cache.concurrent.ConcurrentMapCacheManager}).
 * Позволяет получить набор ключей, «сырой» объект по ключу и доступ к нативной
 * {@link java.util.concurrent.ConcurrentMap}, лежащей под кешом.
 * </p>
 * <p><b>Назначение:</b> упрощение проверок имени/ключа кеша и факта кеширования
 * в unit/интеграционных тестах.</p>
 */

 /**
     * Возвращает множество ключей, присутствующих в кеше.
     *
     * @param cacheManager менеджер кешей
     * @param cacheName    имя кеша (как в {@code @Cacheable(cacheNames=...)})
     * @return набор ключей, сохранённых в указанном кеше
     * @throws IllegalStateException если кеш не является {@code ConcurrentMapCache}
     */

     /**
     * Возвращает «сырой» закешированный объект по ключу.
     * <p>
     * Объект возвращается ровно в том виде, в котором хранится в нативной карте кеша,
     * без дополнительной (де)сериализации.
     * </p>
     *
     * @param cacheManager менеджер кешей
     * @param cacheName    имя кеша
     * @param key          ключ записи в кеше
     * @return значение, ассоциированное с ключом, либо {@code null}, если записи нет
     * @throws IllegalStateException если кеш не является {@code ConcurrentMapCache}
     */

     /**
     * Возвращает нативную {@link java.util.concurrent.ConcurrentMap}, лежащую под кешем.
     * <p>
     * Полезно для низкоуровневых проверок содержимого кеша в тестах
     * (например, сравнение ссылочной равенства объектов).
     * </p>
     *
     * @param cacheManager менеджер кешей
     * @param cacheName    имя кеша
     * @return нативная concurrent-карта кеша
     * @throws IllegalStateException если кеш не является {@link org.springframework.cache.concurrent.ConcurrentMapCache}
     */

     /** Корневой ключ страницы: массив объектов, напр. [{ "items": [ ... ] }]. */
private static final String KEY_ITEMS = "items";

/** Узел элемента внутри страницы: { "item": { ... } }. */
private static final String KEY_ITEM = "item";

/** Массив атрибутов элемента: { "values": [ ... ] }. */
private static final String KEY_VALUES = "values";

/** Узел атрибута: { "attribute": { "slug": "...", ... } }. */
private static final String KEY_ATTRIBUTE = "attribute";

/** Код/идентификатор сущности (банк, атрибут и т.п.). */
private static final String KEY_SLUG = "slug";

/** Значение атрибута (строковое). */
private static final String KEY_VALUE = "value";

/** Человекочитаемое имя сущности. */
private static final String KEY_NAME = "name";


```
