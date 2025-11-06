```java

@Component
@RequiredArgsConstructor
public class AdapterCacheOps {

  public static final String UOM_ALL = "uom_all";
  public static final String MATERIAL_TYPE_ALL = "material_type_all";
  public static final String MATERIAL_ALL = "material_all";

  private final BaseMasterDataRequestService md;
  private final SearchRequestProperties props;
  private final MeasureUnitMapper measureUnitMapper;
  private final MaterialTypeMapper materialTypeMapper;
  private final MaterialMapper materialMapper;

  /** Все UOM. */
  @Cacheable(cacheNames = UOM_ALL, key = "'ALL'", sync = true)
  @Nonnull
  public List<UomBankDto> getAllUoms() {
    var resp = md.requestData(props.getSlugValueForMeasureUnit(), Context.BOOK);
    return createResultWithAttribute(resp, measureUnitMapper);
  }

  /** Все виды ТМЦ. */
  @Cacheable(cacheNames = MATERIAL_TYPE_ALL, key = "'ALL'", sync = true)
  @Nonnull
  public List<MaterialTypeDto> getAllMaterialTypes() {
    var resp = md.requestData(props.getSlugValueForMaterialType(), Context.TMC);
    return createResult(resp, materialTypeMapper);
  }

  /** Все материалы (основная запись). */
  @Cacheable(cacheNames = MATERIAL_ALL, key = "'ALL'", sync = true)
  @Nonnull
  public List<MaterialDto> getAllMaterials() {
    var resp = md.requestData(props.getSlugValueForMaterial(), Context.TMC);
    return createResultWithAttribute(resp, materialMapper);
  }
}


@Slf4j
@Service
@RequiredArgsConstructor
public class AdapterService {

  // cache names
  public static final String UOM_BY_CODE = "uom_by_code";
  public static final String MATERIAL_TYPE_BY_ID = "material_type_by_id";
  public static final String MATERIAL_BY_CODE = "material_by_code";

  private final BatchCacheSupport batchLoad;
  private final BaseMasterDataRequestService md;
  private final SearchRequestProperties props;
  private final MeasureUnitMapper measureUnitMapper;
  private final MaterialTypeMapper materialTypeMapper;
  private final MaterialMapper materialMapper;
  private final AdapterAllCacheOps allOps;

  // ---------- API ----------

  /** UOM: по кодам или все (если список пуст/null). */
  @Nonnull
  public ResultObj<List<UomBankDto>> getUom(@Nullable final List<String> uomCodes) {
    final List<UomBankDto> data = (uomCodes == null || uomCodes.isEmpty())
        ? allOps.getAllUoms()
        : batchLoad.fetchBatch(
            UOM_BY_CODE,
            uomCodes,
            this::loadUomsByCodes,           // miss → один запрос в МД
            UomBankDto::getUomCode,
            UomBankDto.class
          );
    return getSuccessResponse(data);
  }

  /** Вид ТМЦ: по id или все. */
  @Nonnull
  public ResultObj<List<MaterialTypeDto>> getMaterialType(@Nullable final List<String> typeIds) {
    final List<MaterialTypeDto> data = (typeIds == null || typeIds.isEmpty())
        ? allOps.getAllMaterialTypes()
        : batchLoad.fetchBatch(
            MATERIAL_TYPE_BY_ID,
            typeIds,
            this::loadMaterialTypesByIds,
            MaterialTypeDto::getTypeId,
            MaterialTypeDto.class
          );
    return getSuccessResponse(data);
  }

  /** Материалы: по кодам или все. */
  @Nonnull
  public ResultObj<List<MaterialDto>> getMaterial(@Nullable final List<String> materialCodes) {
    final List<MaterialDto> data = (materialCodes == null || materialCodes.isEmpty())
        ? allOps.getAllMaterials()
        : batchLoad.fetchBatch(
            MATERIAL_BY_CODE,
            materialCodes,
            this::loadMaterialsByCodes,
            MaterialDto::getMaterialCode,
            MaterialDto.class
          );
    return getSuccessResponse(data);
  }

  // ---------- Loaders for miss (без кеш-аннотаций) ----------

  @Nonnull
  private List<UomBankDto> loadUomsByCodes(@Nonnull final List<String> codes) {
    final var resp = md.requestDataWithAttribute(
        props.getSlugValueForMeasureUnit(),
        codes,
        Context.BOOK);
    return createResultWithAttribute(resp, measureUnitMapper);
  }

  @Nonnull
  private List<MaterialTypeDto> loadMaterialTypesByIds(@Nonnull final List<String> ids) {
    final var resp = md.requestData(
        props.getSlugValueForMaterialType(),
        ids,
        Context.TMC);
    return createResult(resp, materialTypeMapper);
  }

  @Nonnull
  private List<MaterialDto> loadMaterialsByCodes(@Nonnull final List<String> codes) {
    final var resp = md.requestDataWithAttribute(
        props.getSlugValueForMaterial(),
        codes,
        Context.TMC);
    return createResultWithAttribute(resp, materialMapper);
  }

  // ---------- Invalidation helpers (опционально) ----------
  @CacheEvict(cacheNames = {
      AdapterAllCacheOps.UOM_ALL, UOM_BY_CODE,
      AdapterAllCacheOps.MATERIAL_TYPE_ALL, MATERIAL_TYPE_BY_ID,
      AdapterAllCacheOps.MATERIAL_ALL, MATERIAL_BY_CODE
  }, allEntries = true)
  public void cleanAllCaches() {
    log.info("Adapter caches cleared");
  }
}
```
