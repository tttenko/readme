```java

/**
 * Загрузчик для кеша ставок НДС {@link NdsService2#CACHE_NDS_BY_RATE}.
 * <p>По набору ключей вида {@code <date>} или {@code <date>|<rate>}
 * подгружает данные из МД ({@link BaseMasterDataRequestService}),
 * группирует их по ставкам и формирует бакеты {@link NdsService2.RateBucket}.
 *
 * @see NdsService2
 * @see BatchLoader
 */
@Component
@RequiredArgsConstructor
public class LoaderNdsByRate implements BatchLoader<NdsService2.RateBucket> {

    ...

    /**
     * Имя кеша, для которого работает данный загрузчик.
     *
     * @return имя кеша ставок НДС
     */
    @Override
    public String cacheName() {
        return NdsService2.CACHE_NDS_BY_RATE;
    }

    /**
     * Тип элементов, сохраняемых в кеше.
     *
     * @return класс бакета ставок НДС
     */
    @Override
    public Class<NdsService2.RateBucket> elementType() {
        return NdsService2.RateBucket.class;
    }

    /**
     * Извлекает ключ кеша из значения бакета.
     *
     * @param value бакет со ставками НДС
     * @return строковый ключ кеша
     */
    @Override
    public String extractKey(NdsService2.RateBucket value) {
        return value.key();
    }

    /**
     * Загружает данные из МД по списку ключей кеша.
     * <p>Все ключи должны относиться к одной дате. Для этой даты
     * выполняется запрос в МД, данные группируются по ставкам и
     * далее собираются бакеты для каждого ключа.
     *
     * @param keys список ключей кеша {@code <date>} или {@code <date>|<rate>}
     * @return список бакетов, соответствующих указанным ключам
     */
    @Override
    @NonNull
    public List<NdsService2.RateBucket> fetchByKeys(@NonNull List<String> keys) {
        ...
    }

    /**
     * Извлекает часть даты из ключа кеша.
     * <p>Если символ {@code '|'} отсутствует, весь ключ считается датой.
     *
     * @param key исходный ключ кеша
     * @return строковое представление даты
     */
    private static String extractDate(@NonNull String key) {
        ...
    }

    /**
     * Извлекает ключ ставки из ключа кеша.
     * <p>Если после {@code '|'} нет символов, либо {@code '|'} отсутствует,
     * возвращается специальное значение {@link #ALL_KEY}.
     *
     * @param key исходный ключ кеша
     * @return ключ ставки или {@link #ALL_KEY} для "все ставки"
     */
    private static String extractRateKey(@NonNull String key) {
        ...
    }

    /**
     * Группирует записи НДС по значению ставки.
     *
     * @param all полный список записей НДС, полученных из МД
     * @return отображение «ставка → множество записей с этой ставкой»
     */
    private Map<String, Set<NdsFullDto>> groupByRate(@NonNull List<NdsFullDto> all) {
        ...
    }

    /**
     * Строит список бакетов для заданных ключей кеша.
     *
     * @param keys   исходные ключи кеша
     * @param byRate элементы, сгруппированные по ставкам
     * @param union  объединённое множество всех элементов
     * @return список бакетов, соответствующих каждому ключу
     */
    private List<NdsService2.RateBucket> buildBuckets(
            @NonNull List<String> keys,
            @NonNull Map<String, Set<NdsFullDto>> byRate,
            @NonNull Set<NdsFullDto> union
    ) {
        ...
    }

    /**
     * Возвращает множество элементов НДС для конкретного ключа ставки.
     * <p>Для {@link #ALL_KEY} возвращается объединение всех элементов,
     * для конкретной ставки — элементы из {@code byRate} либо пустое множество.
     *
     * @param rateKey ключ ставки (или {@link #ALL_KEY})
     * @param byRate  элементы, сгруппированные по ставкам
     * @param union   объединённое множество всех элементов
     * @return множество элементов, соответствующих ключу ставки
     */
    private Set<NdsFullDto> resolveItemsForRateKey(
            @NonNull String rateKey,
            @NonNull Map<String, Set<NdsFullDto>> byRate,
            @NonNull Set<NdsFullDto> union
    ) {
        ...
    }

    /**
     * Загружает все записи НДС на указанную дату из мастер-данных.
     *
     * @param dateString дата в строковом формате для запроса
     * @return список записей НДС, актуальных на эту дату
     */
    @NonNull
    private List<NdsFullDto> loadAllFromMd(@NonNull String dateString) {
        ...
    }

    /**
     * Строит критерии поиска по МД для заданной даты.
     * <p>В запрос включаются:
     * <ul>
     *     <li>слуг словаря НДС из {@link SearchRequestProperties#getSlugValueForVat()},</li>
     *     <li>тип ставки, активность, дата начала и окончания действия ставки.</li>
     * </ul>
     *
     * @param dateString дата, по которой отбираются ставки, в строковом виде
     * @return объект критериев поиска для {@link BaseMasterDataRequestService}
     */
    @NonNull
    private ItemsSearchCriteriaRequest buildRequest(@NonNull String dateString) {
        ...
    }
}
```
