```java
/**
 * Batch-loader для подгрузки справочника регионов по коду региона из АС Master Data.
 * <p>
 * Используется батчевой инфраструктурой кэша (см. {@link BatchLoader}) для получения набора {@link RegionDto}
 * по списку ключей (кодов регионов) и последующего сохранения результата в кэш
 * {@link RegionService#REGION_BY_CODE}.
 * <p>
 * Источник данных: Master Data reference-book, идентификатор справочника берётся из настроек
 * {@link SearchRequestProperties#getSlugValueForRegion()}, поиск выполняется по полю slug/коду региона
 * (передаётся список {@code keys}) в контексте {@link SearchRequestProperties.Context#BOOK}.
 * <p>
 * Если список ключей пустой, запрос в Master Data не выполняется (возвращается пустой список).
 *
 * @author
 * @see RegionService
 * @see RegionCacheOps
 * @see BaseMasterDataRequestService
 * @see SearchRequestProperties
 * @since 1.0
 */
@Component
@RequiredArgsConstructor
public class LoaderRegionByCode implements BatchLoader<RegionDto> {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final RegionMapper regionMapper;

    /**
     * Возвращает имя кэша, который обслуживает данный loader.
     *
     * @return имя кэша для поиска регионов по коду
     * @see RegionService#REGION_BY_CODE
     */
    @Override
    public String cacheName() {
        return RegionService.REGION_BY_CODE;
    }

    /**
     * Возвращает тип элементов, которые возвращает loader.
     *
     * @return класс {@link RegionDto}
     */
    @Override
    public Class<RegionDto> elementType() {
        return RegionDto.class;
    }

    /**
     * Извлекает ключ кэширования из DTO региона.
     *
     * @param value DTO региона (не {@code null})
     * @return код региона (используется как ключ кэша)
     */
    @Override
    public String extractKey(RegionDto value) {
        return value.getRegionCode();
    }

    /**
     * Загружает регионы из Master Data по списку кодов регионов.
     * <p>
     * При пустом списке ключей возвращает пустой список и не обращается к backend.
     *
     * @param keys список кодов регионов (не {@code null})
     * @return список найденных {@link RegionDto} (может быть пустым)
     *
     * @implNote Делегирует вызов в {@link BaseMasterDataRequestService#requestData(String, List, SearchRequestProperties.Context)}
     * и маппит результат через {@link RegionMapper#mapItemToDto(java.util.Map)} (метод-референс).
     *
     * @throws RuntimeException если Master Data вернул неуспешный статус (semantic != "S")
     *                          или произошла ошибка запроса/маппинга ответа
     */
    @Override
    public List<RegionDto> fetchByKeys(List<String> keys) {
        if (keys.isEmpty()) {
            return List.of();
        }

        GetItemsSearchResponse response = baseMasterDataRequestService.requestData(
            properties.getSlugValueForRegion(),
            keys,
            SearchRequestProperties.Context.BOOK
        );

        return BaseMasterDataRequestService.createResult(response, regionMapper::mapItemToDto);
    }
}
```
