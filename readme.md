```java

/**
 * Batch-загрузчик валют по их коду для использования в {@link CacheGetOrLoadService}.
 * <p>
 * Реализует контракт {@link BatchLoader}:
 * <ul>
 *   <li>знает имя кеша, к которому привязан ({@link #cacheName()});</li>
 *   <li>определяет тип элементов кеша ({@link #elementType()});</li>
 *   <li>умеет извлекать ключ кеширования из доменного объекта ({@link #extractKey(CurrencyDto)});</li>
 *   <li>умеет батчом подгружать данные по списку ключей ({@link #fetchByKeys(List)}).</li>
 * </ul>
 */
@Component
@RequiredArgsConstructor
public class LoaderCurrencyByCode implements BatchLoader<CurrencyDto> {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final CurrencyMapper currencyMapper;

    /**
     * Имя кеша, с которым работает данный лоадер.
     * <p>
     * Должно совпадать с именем, под которым кеш регистрируется в
     * {@link org.springframework.cache.CacheManager} и которое использует
     * {@link CurrencyService2}.
     */
    @Override
    public String cacheName() {
        return CurrencyService2.CURRENCY_BY_CODE;
    }

    /**
     * Тип элементов, которые возвращает лоадер и которые будут храниться в кеше.
     */
    @Override
    public Class<CurrencyDto> elementType() {
        return CurrencyDto.class;
    }

    /**
     * Извлекает ключ кеширования из доменного объекта валюты.
     * <p>
     * Здесь в качестве ключа используется код валюты.
     *
     * @param value объект валюты
     * @return строковый ключ для кеша (код валюты)
     */
    @Override
    public String extractKey(CurrencyDto value) {
        return value.getCurrencyCode();
    }

    /**
     * Загружает список валют из мастер-данных по переданным кодам.
     * <p>
     * Метод вызывается {@link CacheGetOrLoadService}, когда в кеше отсутствуют
     * значения для запрошенных ключей. Возвращённые объекты будут сохранены в кеш
     * под ключами, полученными через {@link #extractKey(CurrencyDto)}.
     *
     * @param keys список кодов валют (ключи кеша)
     * @return список найденных валют; пустой список, если список ключей пуст
     */
    @Override
    @NonNull
    public List<CurrencyDto> fetchByKeys(@NonNull List<String> keys) {
        if (keys.isEmpty()) {
            return List.of();
        }

        GetItemsSearchResponse resp =
                baseMasterDataRequestService.requestDataByAttributes(
                        properties.getSlugValueForCurrency(),
                        properties.getCurrencyAttributeId(),
                        keys
                );

        return BaseMasterDataRequestService.createResultWithAttribute(resp, currencyMapper);
    }
}
```
