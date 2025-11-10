```java

/**
 * Компонент, отвечающий за кэширование справочников мастер-данных (UOM, MaterialType, Material).
 * <p>
 * Содержит методы для загрузки всех элементов справочников с кэшированием
 * и выборочной загрузки данных без кэширования.
 * </p>
 *
 * @author Максим Коптенко
 */
@Component
@RequiredArgsConstructor
public class AdapterCacheOps {

    /**
     * Загружает все единицы измерения (UOM) и сохраняет результат в кэш.
     *
     * @return список всех {@link UomBankDto}, полученных из мастер-данных.
     */
    @Cacheable(cacheNames = UOM_ALL, key = "'ALL'", sync = true)
    @NonNull
    public List<UomBankDto> getAllUoms() {
        // ...
    }

    /**
     * Загружает все типы материалов и сохраняет результат в кэш.
     *
     * @return список всех {@link MaterialTypeDto}, полученных из мастер-данных.
     */
    @Cacheable(cacheNames = MATERIAL_TYPE_ALL, key = "'ALL'", sync = true)
    @NonNull
    public List<MaterialTypeDto> getAllMaterialTypes() {
        // ...
    }

    /**
     * Загружает все материалы и сохраняет результат в кэш.
     *
     * @return список всех {@link MaterialDto}, полученных из мастер-данных.
     */
    @Cacheable(cacheNames = MATERIAL_ALL, key = "'ALL'", sync = true)
    @NonNull
    public List<MaterialDto> getAllMaterials() {
        // ...
    }

    /**
     * Загружает единицы измерения (UOM) по списку кодов без кэширования.
     *
     * @param codes список кодов единиц измерения.
     * @return список {@link UomBankDto}, соответствующих указанным кодам.
     */
    @NonNull
    public List<UomBankDto> loadUomsByCodes(@NonNull final List<String> codes) {
        // ...
    }

    /**
     * Загружает типы материалов по их идентификаторам без кэширования.
     *
     * @param ids список идентификаторов типов материалов.
     * @return список {@link MaterialTypeDto}, соответствующих указанным идентификаторам.
     */
    @NonNull
    public List<MaterialTypeDto> loadMaterialTypesByIds(@NonNull final List<String> ids) {
        // ...
    }

    /**
     * Загружает материалы по списку кодов без кэширования.
     *
     * @param codes список кодов материалов.
     * @return список {@link MaterialDto}, соответствующих указанным кодам.
     */
    @NonNull
    public List<MaterialDto> loadMaterialsByCodes(@NonNull final List<String> codes) {
        // ...
    }
}

```
