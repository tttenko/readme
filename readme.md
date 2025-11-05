```java

/**
 * Интеграционные/компонентные тесты кеширования в {@link NdsService2}.
 * <p>
 * Используется реальный {@link CaffeineCacheManager} и вспомогательный
 * {@link BatchCacheSupport} c моками зависимостей сервиса.
 *
 * <h2>Что проверяем</h2>
 * <ul>
 *   <li>прогрев кеша и последующие cache-hit по ключу ставки;</li>
 *   <li>создание "общего" бакета {@code __ALL__} при отсутствии фильтра по ставкам;</li>
 *   <li>добавление нового бакета при запросе с новой ставкой;</li>
 *   <li>ручную очистку кеша и повторный прогрев.</li>
 * </ul>
 *
 * @see NdsService2
 * @see RateBucket
 * @since 1.0
 */
class NdsService2CacheTest {
java
Копировать код
    /**
     * Инициализация окружения теста.
     * <p><b>Given</b>: чистый инстанс {@link CaffeineCacheManager} и моки зависимостей.<br>
     * <b>When</b>: собираем {@link NdsService2} и подставляем {@link BatchCacheSupport}.<br>
     * <b>Then</b>: кеш {@code nds_by_rate} очищен, моки подготовлены.
     */
    @BeforeEach
    void setUp() { ... }
java
Копировать код
    /**
     * Прогрев → cache-hit → ручная очистка.
     * <p><b>Given</b>: одна ставка НДС и элементы из МД.<br>
     * <b>When</b>: первый вызов прогревает кеш, второй — попадает в кеш, затем выполняем clear.<br>
     * <b>Then</b>: первый вызов обращается в МД, второй — нет; после clear повторный вызов снова идет в МД.
     *
     * @implNote Проверяется состав бакета по ключу ставки и количество обращений к {@code baseService}.
     */
    @Test
    @DisplayName("Warm-up → ключ появляется; повторный вызов ⇒ hit; очистка вручную очищает кеш")
    void warmup_hit_manualClear() { ... }
java
Копировать код
    /**
     * Без параметра rate создаётся только общий бакет {@code __ALL__}.
     * <p><b>Given</b>: два элемента с разными ставками, запрос без списка ставок.<br>
     * <b>When</b>: вызываем {@link NdsService2#getBasicVatRate}.<br>
     * <b>Then</b>: в кеше появляется только ключ {@code __ALL__}, в бакете — оба id.
     */
    @Test
    @DisplayName("Без параметра rate создаётся бакет '__ALL__'")
    void no_rate_creates_all_bucket() { ... }
java
Копировать код
    /**
     * Новый rate добавляет новый бакет в кеше.
     * <p><b>Given</b>: сначала запрашиваем ставку 5, затем ставку 10.<br>
     * <b>When</b>: два последовательных вызова {@link NdsService2#getBasicVatRate}.<br>
     * <b>Then</b>: в кеше есть оба ключа (5 и 10), содержимое бакетов соответствует id элементов.
     */
    @Test
    @DisplayName("Новый rate ⇒ добавляется новый бакет")
    void new_rate_adds_new_bucket() { ... }
java
Копировать код
    /**
     * Возвращает содержимое бакета из кеша по ключу ставки.
     *
     * @param key ключ бакета (ставка НДС или {@code __ALL__})
     * @return {@link RateBucket} или {@code null}, если бакета нет/кеш не создан
     * @implNote Утилитный метод для ассертов в тестах.
     */
    private RateBucket getBucket(String key) { ... }
}






```
