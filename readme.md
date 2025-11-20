```java

@Component
@RequiredArgsConstructor
public class LoaderUomByCode implements BatchLoader<UomBankDto> {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final MeasureUnitMapper measureUnitMapper;

    /** Имя кэша UOM, с которым работает лоадер. */
    @Override
    public String cacheName() {
        return AdapterService2.UOM_BY_CODE;
    }

    /** Тип элементов, которые возвращает лоадер и которые хранятся в кэше. */
    @Override
    public Class<UomBankDto> elementType() {
        return UomBankDto.class;
    }

    /** Извлекает ключ кэширования из DTO единицы измерения. */
    @Override
    public String extractKey(UomBankDto value) {
        return value.getUomCode();
    }

    /** Загружает список UOM из мастер-данных по переданным кодам. */
    @Override
    @NonNull
    public List<UomBankDto> fetchByKeys(@NonNull List<String> keys) {
        if (keys.isEmpty()) {
            return List.of();
        }

        final GetItemsSearchResponse resp =
                baseMasterDataRequestService.requestDataWithAttribute(
                        properties.getSlugValueForMeasureUnit(),
                        keys,
                        SearchRequestProperties.Context.BOOK
                );

        return BaseMasterDataRequestService.createResultWithAttribute(resp, measureUnitMapper);
    }
}

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

```
