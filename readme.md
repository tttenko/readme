```java

/**
 * Batch-загрузчик типов материалов по их идентификаторам.
 * Привязан к кэшу {@link AdapterService2#MATERIAL_TYPE_BY_ID} и
 * загружает данные из мастер-данных в виде {@link MaterialTypeDto}.
 */
@Component
@RequiredArgsConstructor
public class LoaderMaterialTypeById implements BatchLoader<MaterialTypeDto> {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final MaterialTypeMapper materialTypeMapper;

    /** Имя кэша типов материалов, с которым работает лоадер. */
    @Override
    public String cacheName() {
        return AdapterService2.MATERIAL_TYPE_BY_ID;
    }

    /** Тип элементов, которые возвращает лоадер и которые хранятся в кэше. */
    @Override
    public Class<MaterialTypeDto> elementType() {
        return MaterialTypeDto.class;
    }

    /** Извлекает ключ кэширования из DTO типа материала. */
    @Override
    public String extractKey(MaterialTypeDto value) {
        return value.getTypeId();
    }

    /** Загружает типы материалов из мастер-данных по их идентификаторам. */
    @Override
    @NonNull
    public List<MaterialTypeDto> fetchByKeys(@NonNull List<String> keys) {
        if (keys.isEmpty()) {
            return List.of();
        }

        final GetItemsSearchResponse resp =
                baseMasterDataRequestService.requestData(
                        properties.getSlugValueForMaterialType(),
                        keys,
                        SearchRequestProperties.Context.TMC
                );

        return BaseMasterDataRequestService.createResult(resp, materialTypeMapper);
    }
}

/**
 * Batch-загрузчик материалов по их кодам.
 * Используется кэш {@link AdapterService2#MATERIAL_BY_CODE};
 * запрашивает данные из мастер-данных и маппит их в {@link MaterialDto}.
 */
 @Component
@RequiredArgsConstructor
public class LoaderMaterialByCode implements BatchLoader<MaterialDto> {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final MaterialMapper materialMapper;

    /** Имя кэша материалов, с которым работает лоадер. */
    @Override
    public String cacheName() {
        return AdapterService2.MATERIAL_BY_CODE;
    }

    /** Тип элементов, которые возвращает лоадер и которые хранятся в кэше. */
    @Override
    public Class<MaterialDto> elementType() {
        return MaterialDto.class;
    }

    /** Извлекает ключ кэширования из DTO материала. */
    @Override
    public String extractKey(MaterialDto value) {
        return value.getMaterialCode();
    }

    /** Загружает материалы из мастер-данных по списку кодов. */
    @Override
    @NonNull
    public List<MaterialDto> fetchByKeys(@NonNull List<String> keys) {
        if (keys.isEmpty()) {
            return List.of();
        }

        final GetItemsSearchResponse resp =
                baseMasterDataRequestService.requestDataWithAttribute(
                        properties.getSlugValueForMaterial(),
                        keys,
                        SearchRequestProperties.Context.TMC
                );

        return BaseMasterDataRequestService.createResultWithAttribute(resp, materialMapper);
    }
}




@Slf4j
@Service
@RequiredArgsConstructor
public class AdapterService2 {

    public static final String UOM_BY_CODE = "uom_by_code";
    public static final String MATERIAL_TYPE_BY_ID = "material_type_by_id";
    public static final String MATERIAL_BY_CODE = "material_by_code";

    private final AdapterCacheOps adapterCacheOps;
    private final CacheGetOrLoadService cacheGetOrLoadService;

    /** Возвращает список единиц измерения (UOM) по их кодам или все UOM, если список пустой. */
    @NonNull
    public ResultObj<List<UomBankDto>> getUom(@Nullable final List<String> uomCodes) {
        final boolean requestAll = (uomCodes == null || uomCodes.isEmpty());

        final List<UomBankDto> data = requestAll
                ? adapterCacheOps.getAllUoms()
                : cacheGetOrLoadService.fetchData(UOM_BY_CODE, uomCodes);

        return getSuccessResponse(data);
    }

    /** Возвращает список типов материалов по их идентификаторам или все типы, если список пустой. */
    @NonNull
    public ResultObj<List<MaterialTypeDto>> getMaterialType(@Nullable final List<String> typeIds) {
        final boolean requestAll = (typeIds == null || typeIds.isEmpty());

        final List<MaterialTypeDto> data = requestAll
                ? adapterCacheOps.getAllMaterialTypes()
                : cacheGetOrLoadService.fetchData(MATERIAL_TYPE_BY_ID, typeIds);

        return getSuccessResponse(data);
    }

    /** Возвращает список материалов по их кодам или все материалы, если список пустой. */
    @NonNull
    public ResultObj<List<MaterialDto>> getMaterial(@Nullable final List<String> materialCodes) {
        final boolean requestAll = (materialCodes == null || materialCodes.isEmpty());

        final List<MaterialDto> data = requestAll
                ? adapterCacheOps.getAllMaterials()
                : cacheGetOrLoadService.fetchData(MATERIAL_BY_CODE, materialCodes);

        return getSuccessResponse(data);
    }
}

```
