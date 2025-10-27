```java

/**
 * Тесты batch-кеширования поверх {@link BatchCacheSupport} с {@code ConcurrentMapCacheManager}.
 *
 * <p>Проверяем:
 * <ul>
 *   <li>батч-догрузку без N+1 (один вызов loader на множество промахов);</li>
 *   <li>hit/miss через счётчик вызовов loader;</li>
 *   <li>отсутствие дублей в кеше (одна запись на ключ);</li>
 *   <li>корректную работу с дубликатами ключей, «лишними» элементами и {@code null} из loader;</li>
 *   <li>конкурентные батчи с пересекающимися ключами.</li>
 * </ul>
 * </p>
 *
 * @implNote В юнитах используем {@code ConcurrentMapCacheManager} — быстро и детерминированно.
 * Провайдер-специфику (TTL, метрики Caffeine) имеет смысл проверять отдельным IT.
 */

/**
 * Минимальный DTO для тестов кеша.
 *
 * <p>Ключом служит поле {@code code}. Равенство/хеш не переопределяем — для тестов это не требуется.</p>
 */


/**
 * Инициализирует {@link ConcurrentMapCacheManager} c allowNullValues и
 * очищает нативную карту кеша перед каждым тестом.
 */


givenSomeKeysCached_whenFetchBatch_thenOnlyMissesLoaded_andNoDuplicatesInCache
/**
 * <b>Given</b> в кеше уже есть {@code A}, запрашиваем {@code [A,B,C]}.<br/>
 * <b>When</b> вызываем {@code fetchBatch}.<br/>
 * <b>Then</b> loader вызывается ровно один раз с промахами {@code [B,C]}; в кеше лежат
 * уникальные ключи {@code A,B,C}; результат содержит три элемента.
 *
 * @see CacheIntrospection#keys
 * @see org.mockito.ArgumentCaptor
 */

givenAllKeysCached_whenFetchBatch_thenLoaderNotCalled_andPureHit
/**
 * <b>Given</b> в кеше уже есть {@code A} и {@code B}.<br/>
 * <b>When</b> запрашиваем {@code [A,B]}.<br/>
 * <b>Then</b> чистый hit: loader не вызывается, размер результата равен 2,
 * а состав ключей в кеше не меняется.
 */

givenDuplicateKeys_whenFetchBatch_thenLoaderGetsDistinctMisses_andCacheStoresSingleEntryPerKey
/**
 * <b>Given</b> запрос содержит дубликаты ключей {@code [A,A,B,B]}.<br/>
 * <b>When</b> выполняем batch-загрузку.<br/>
 * <b>Then</b> loader получает <i>уникальные</i> промахи ({@code [A,B]}) ровно один раз,
 * а в кеше по каждому ключу ровно одна запись; результат содержит как минимум {@code A} и {@code B}.
 */

givenLoaderReturnsExtraOrMissing_whenFetchBatch_thenCacheOnlyRequestedAndFoundItems
/**
 * <b>Given</b> запрошены {@code [A,B]}, loader возвращает найденный {@code A} и «лишний» {@code X}.<br/>
 * <b>When</b> выполняем {@code fetchBatch}.<br/>
 * <b>Then</b> в кеше оказываются {@code A} и {@code X} (семантика текущей реализации — кэшируем всё, что пришло),
 * а в результате возвращается только пересечение «запрошено ∧ найдено» — {@code A}.
 *
 * @implNote Если потребуется не кэшировать «лишние» ключи, измените прод-код:
 * фильтровать загруженные элементы по множеству запрошенных ключей.
 */

givenNullsFromLoader_whenFetchBatch_thenNullIgnored_andNoNullInCache
/**
 * <b>Given</b> loader возвращает список, содержащий {@code null}.<br/>
 * <b>When</b> выполняем batch-загрузку.<br/>
 * <b>Then</b> {@code null}-элементы игнорируются: в кеше нет записей для них,
 * результат содержит только валидные объекты.
 *
 * @implNote Требует null-safe реализации: пропускать {@code null} и пустые ключи.
 */

givenOverlappingConcurrentBatches_whenFetchBatch_thenNoNPlusOne_andFewLoaderCallsOnly
/**
 * <b>Given</b> два конкурентных запроса с пересечением ключей
 * {@code [A,B,C]} и {@code [B,C,D]}.<br/>
 * <b>When</b> запускаем оба батча параллельно.<br/>
 * <b>Then</b> нет N+1: loader вызывается не чаще количества конкурентных батчей
 * (точно не по одному разу на ключ); в кеше — уникальные {@code A,B,C,D}.
 *
 * @throws Exception при таймауте ожидания потоков
 */




```
