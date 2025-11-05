```java

/**
 * Unit-тесты для {@link NdsService2} без поднятия Spring-контекста.
 *
 * <p>Стратегия: чистый Mockito (@ExtendWith(MockitoExtension)), все зависимости
 * сервиса замоканы. Через {@link org.mockito.ArgumentCaptor} перехватываются
 * ключи батч-кеша и собранный запрос к МД.</p>
 *
 * <ul>
 *   <li>Проверка выбора ключей для batch-кеша.</li>
 *   <li>Проверка сборки {@link ItemsSearchCriteriaRequest} из properties и даты.</li>
 *   <li>Проверка фильтрации по окнам дат/кодам/ставкам и маппинга в {@code NdsDto}.</li>
 * </ul>
 *
 * @see NdsService2
 * @since 1.0
 */
class NdsService2Test {
java
Копировать код
/**
 * GIVEN: параметр {@code rate} не задан (null/empty).<br>
 * WHEN: вызываем {@link NdsService2#getBasicVatRate}.<br>
 * THEN: в {@code BatchCacheSupport.fetchBatch(...)} уходит единственный
 * miss-ключ {@code "__ALL__"}, а метод возвращает непустой результат.
 *
 * <p>Через {@link org.mockito.ArgumentCaptor} проверяется список ключей,
 * с которым сервис обращается к batch-кешу.</p>
 */
@Test
@DisplayName("Если rate=null/empty → запрашиваем из кеша спец-ключ __ALL__")
void keys_whenRateNull_thenAllKey() { ... }
java
Копировать код
/**
 * GIVEN: в запрос передан список ставок {@code rate}.<br>
 * WHEN: вызываем {@link NdsService2#getBasicVatRate}.<br>
 * THEN: набор miss-ключей, переданный в {@code BatchCacheSupport.fetchBatch}, в точности
 * совпадает с переданными ставками (порядок не важен).
 */
@Test
@DisplayName("Если rate задан → ключи в батче совпадают со значениями rate")
void keys_whenRateProvided_thenExactKeys() { ... }
java
Копировать код
/**
 * GIVEN: произвольная дата запроса; в {@code properties} настроены id атрибутов и slug словаря.<br>
 * WHEN: вызываем {@link NdsService2#getBasicVatRate}.<br>
 * THEN: в {@link BaseMasterDataRequestService#requestData} уходит
 * {@link ItemsSearchCriteriaRequest}, собранный из значений properties, с корректно
 * отформатированной датой. Поля {@code dictionaryName} и {@code reference} соответствуют
 * значениям properties.
 *
 * <p>Через {@link org.mockito.ArgumentCaptor} перехватывается и проверяется целиком
 * объект запроса.</p>
 */
@Test
@DisplayName("buildRequest: в requestData уходит словарь из properties и присутствует дата запроса")
void buildRequest_shouldPassDictionaryAndDate() { ... }
java
Копировать код
/**
 * GIVEN: в источнике есть элементы с разными окнами начала/окончания действия,
 * кодами и ставками (часть подходит под фильтр, часть нет).<br>
 * WHEN: вызываем {@link NdsService2#getBasicVatRate} с конкретной датой и фильтрами
 * по коду/ставке.<br>
 * THEN: в результате остаются только элементы, удовлетворяющие предикатам
 * {@code isBefore}/{@code isAfter} и совпадающие по коду/ставке; при этом они
 * замаплены в {@code NdsDto} и возвращаются в исходном порядке ключей.
 */
@Test
@DisplayName("Фильтр/маппинг: возвращаем только подходящее и уже в виде DTO")
void filterAndMapping_shouldReturnOnlyMatchedAndMapped() { ... }
java
Копировать код
/**
 * Удобный билдер тестовых {@code NdsFullDto}.
 *
 * @param id    идентификатор
 * @param rate  ставка НДС
 * @param code  код
 * @param start дата начала действия (включительно)
 * @param end   дата окончания действия (исключая), может быть {@code null}
 * @return сконструированный {@code NdsFullDto} для сценариев теста
 */
private static NdsFullDto nds(String id, String rate, String code,
                              ZonedDateTime start, ZonedDateTime end) { ... }

```
