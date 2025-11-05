```java

/**
 * Возвращает базовые ставки НДС, применяя фильтры по дате, коду и ставке.
 *
 * @param date дата, на которую проверяется актуальность; если null — берётся now()
 * @param code список кодов для фильтрации (nullable)
 * @param rate список ставок для фильтрации; если null/пусто — запрашивается бакет "__ALL__"
 * @return успешный результат с DTO ставок НДС
 */
public ResultObj<List<NdsDto>> getBasicVatRate(@Nullable ZonedDateTime date,
                                               @Nullable List<String> code,
                                               @Nullable List<String> rate) { ... }


/**
 * Загружает из МД все записи по НДС на указанную дату.
 *
 * @param date дата фильтрации
 * @return список полных DTO из мастер-данных
 */
private List<NdsFullDto> loadAllFromMd(@Nonnull ZonedDateTime date) { ... }


/**
 * Делит загруженные элементы на «бакеты» по ключам ставок.
 * Для ключа "__ALL__" кладёт объединение всех элементов.
 *
 * @param all      все записи, загруженные из МД
 * @param missKeys ключи, для которых не нашлось данных в кеше
 * @return список бакетов (ключ + набор элементов)
 */
private List<RateBucket> partitionByRate(@Nonnull List<NdsFullDto> all,
                                         @Nonnull List<String> missKeys) { ... }


/**
 * Превращает бакеты в плоский список DTO с применением фильтров.
 *
 * @param buckets   бакеты элементов
 * @param dateTime  дата для фильтра по периодам действия
 * @param rate      фильтр по ставке (nullable)
 * @param code      фильтр по коду (nullable)
 * @return список DTO для ответа
 */
private List<NdsDto> mapBucketsToDtoList(@Nonnull List<RateBucket> buckets,
                                         @Nonnull ZonedDateTime dateTime,
                                         @Nullable List<String> rate,
                                         @Nullable List<String> code) { ... }

/**
 * Композиция предикатов фильтрации для элемента НДС:
 * период действия, совпадение по ставке/коду при наличии фильтров.
 *
 * @param date дата проверки
 * @param rate список ставок для фильтрации (nullable)
 * @param code список кодов для фильтрации (nullable)
 * @return предикат фильтрации для Stream API
 */
private Predicate<NdsFullDto> matches(@Nonnull ZonedDateTime date,
                                      @Nullable List<String> rate,
                                      @Nullable List<String> code) { ... }

/**
 * Маппинг полного элемента НДС в короткий DTO для ответа.
 *
 * @param e полный DTO из МД
 * @return упрощённый DTO
 */
private NdsDto toDto(@Nonnull NdsFullDto e) { ... }

/**
 * Строит критерии поиска в МД под заданную дату: тип ставки, активность,
 * границы периода (start <= date <= end).
 *
 * @param dateString дата в формате {@code yyyy-MM-dd'T'HH:mmXXX}
 * @return запрос критериев для BaseMasterDataRequestService
 */
private ItemsSearchCriteriaRequest buildRequest(@Nonnull String dateString) { ... }

/**
 * Проверяет, что окончание периода элемента позже заданной даты
 * или не задано вовсе (открытый период).
 *
 * @param date дата проверки
 * @param e    элемент НДС
 * @return true, если элемент актуален по окончанию периода
 */
private static boolean isAfter(@Nonnull ZonedDateTime date, @Nonnull NdsFullDto e) { ... }

/**
 * Проверяет, что начало периода элемента раньше заданной даты
 * (начало задано и уже наступило).
 *
 * @param date дата проверки
 * @param e    элемент НДС
 * @return true, если элемент актуален по началу периода
 */
private static boolean isBefore(@Nonnull ZonedDateTime date, @Nonnull NdsFullDto e) { ... }

/**
 * Полностью очищает кеш бакетов ставок НДС.
 * Используется для принудительного сброса перед прогревом/тестами.
 */
@CacheEvict(cacheNames = CACHE_NDS_BY_RATE, allEntries = true)
public void cleanCache() { ... }

```
