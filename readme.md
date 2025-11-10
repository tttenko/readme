```java

/**
 * Сервис для получения данных из мастер-данных с поддержкой кэширования.
 * <p>
 * Делегирует загрузку справочников классу {@link AdapterCacheOps} и использует
 * {@link BatchCacheSupport} для пакетной подгрузки данных по ключам.
 * </p>
 * Сервис обрабатывает:
 * <ul>
 *   <li>Единицы измерения (UOM)</li>
 *   <li>Типы материалов</li>
 *   <li>Материалы</li>
 * </ul>
 *
 * @author Максим Коптенко
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AdapterService2 {

    /**
     * Код операции для единиц измерения.
     */
    public static final String UOM_BY_CODE = "uom_by_code";

    /**
     * Код операции для типов материалов.
     */
    public static final String MATERIAL_TYPE_BY_ID = "material_type_by_id";

    /**
     * Код операции для материалов.
     */
    public static final String MATERIAL_BY_CODE = "material_by_code";

    private final BatchCacheSupport batchLoad;
    private final AdapterCacheOps adapterCacheOps;

    /**
     * Возвращает список единиц измерения (UOM) по их кодам.
     * <p>
     * Если список кодов пустой или равен {@code null}, метод возвращает все единицы
     * измерения из кэша. Иначе — выполняет пакетную загрузку данных.
     * </p>
     *
     * @param uomCodes список кодов единиц измерения (может быть {@code null}).
     * @return результат запроса, обёрнутый в {@link ResultObj}, содержащий список {@link UomBankDto}.
     */
    @NonNull
    public ResultObj<List<UomBankDto>> getUom(@Nullable final List<String> uomCodes) {
        // ...
    }

    /**
     * Возвращает список типов материалов по их идентификаторам.
     * <p>
     * Если список идентификаторов пустой или равен {@code null}, возвращаются все типы
     * материалов из кэша. Иначе выполняется загрузка по идентификаторам.
     * </p>
     *
     * @param typeIds список идентификаторов типов материалов (может быть {@code null}).
     * @return результат запроса, содержащий список {@link MaterialTypeDto}.
     */
    @NonNull
    public ResultObj<List<MaterialTypeDto>> getMaterialType(@Nullable final List<String> typeIds) {
        // ...
    }

    /**
     * Возвращает список материалов по их кодам.
     * <p>
     * Если список кодов пустой или равен {@code null}, метод возвращает все материалы из кэша.
     * При наличии кодов выполняется пакетная загрузка данных.
     * </p>
     *
     * @param materialCodes список кодов материалов (может быть {@code null}).
     * @return результат запроса, содержащий список {@link MaterialDto}.
     */
    @NonNull
    public ResultObj<List<MaterialDto>> getMaterial(@Nullable final List<String> materialCodes) {
        // ...
    }
}

```
