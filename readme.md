```java

/**
 * Утилитарный сервис поверх Spring Cache для батч-работы с кэшем.
 * <p>Отвечает за три операции:
 * <ul>
 *   <li>определение «промахов» по ключам ({@link #collectMisses});</li>
 *   <li>запись загруженных элементов в кэш ({@link #putToCache});</li>
 *   <li>чтение значений из кэша ({@link #readFromCache}).</li>
 * </ul>
 *
 * <h3>Контракты</h3>
 * <ul>
 *   <li>Класс статлесс и потокобезопасен.</li>
 *   <li>Не кэширует {@code null}-значения; пустые/blank ключи запрещены.</li>
 *   <li>Если кэш с именем не найден — выбрасывается {@code MdaCacheNotFoundException} (fail-fast).</li>
 *   <li>Если из элемента извлечён пустой ключ — выбрасывается {@code MdaInvalidCacheKeyException}.</li>
 *   <li>{@link #collectMisses} возвращает уникальные ключи (без гарантии порядка);</li>
 *       {@link #readFromCache} сохраняет порядок входных ключей и отфильтровывает {@code null}.</li>
 * </ul>
 *
 * @see org.springframework.cache.CacheManager
 * @see org.springframework.cache.Cache
 * @see #collectMisses(String, java.util.List, Class)
 * @see #putToCache(String, java.util.List, java.util.function.Function)
 * @see #readFromCache(String, java.util.List, Class)
 */
```
