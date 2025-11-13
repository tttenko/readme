```java

/**
 * Возвращает список ключей-«промахов» (в кэше нет значения).
 *
 * @param cacheName имя кэша
 * @param keys      запрошенные ключи (может быть пустым)
 * @param type      тип значений в кэше
 * @param <T>       тип значения
 * @return уникальные ключи, для которых cache.get(key, type) вернул {@code null}
 * @throws MdaCacheNotFoundException если кэш с именем {@code cacheName} не найден
 */
public <T> List<String> collectMisses(String cacheName, List<String> keys, Class<T> type) { ... }
java
Копировать код
/**
 * Кладёт загруженные элементы в кэш.
 * <p>Элементы с null/blank ключом не допускаются.</p>
 *
 * @param cacheName   имя кэша
 * @param loaded      элементы для сохранения (null-элементы игнорируются)
 * @param keyExtractor функция извлечения ключа из элемента
 * @param <T>         тип элемента
 * @throws MdaCacheNotFoundException   если кэш не найден
 * @throws MdaInvalidCacheKeyException если извлечённый ключ null/blank
 */
public <T> void putToCache(String cacheName, List<T> loaded, Function<T, String> keyExtractor) { ... }
java
Копировать код
/**
 * Читает значения из кэша по списку ключей.
 * <p>Порядок результата соответствует порядку ключей, null-значения отфильтровываются.</p>
 *
 * @param cacheName имя кэша
 * @param keys      ключи для чтения
 * @param type      тип значений
 * @param <T>       тип значения
 * @return список найденных значений без {@code null}
 * @throws MdaCacheNotFoundException если кэш не найден
 */
public <T> List<T> readFromCache(String cacheName, List<String> keys, Class<T> type) { ... }
java
Копировать код
/**
 * Возвращает кэш по имени или выбрасывает исключение.
 *
 * @param cacheName имя кэша
 * @return кэш
 * @throws MdaCacheNotFoundException если кэш не найден
 */
private Cache getCacheOrThrowException(String cacheName) { ... }
```
