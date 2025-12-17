```java


 @Slf4j
@Validated
@RestController
@RequiredArgsConstructor
@RequestMapping(value = "/api/v1/info/region_code", produces = APPLICATION_JSON_UTF8_VALUE)
@Tag(
    name = "Region controller",
    description = "Сервис получения перечня регионов из АС Мастер-данные"
)
public class RegionCodeController {

  private final RegionService regionService;
  private final ResponseHandler responseHandler;

  /** Ищет регионы по списку кодов региона (slug) */
  @GetMapping
  @Operation(
      operationId = "searchRegion",
      summary = "Получение перечня регионов по массиву кодов регионов"
  )
  @ApiResponses({
      @ApiResponse(responseCode = "200", description = "Успешный поиск регионов"),
      @ApiResponse(
          responseCode = "400",
          description = "Некорректные параметры запроса (валидация, пустой/отсутствующий regionCode)"
      ),
      @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
  })
  public ResultObj<List<RegionDto>> searchRegionsByCode(
      @RequestParam(name = "regionCode")
      @NotEmpty(message = "Параметр regionCode должен содержать хотя бы одно значение")
      List<@NotBlank(message = "Код региона не должен быть пустым") String> regionCodes
  ) {
    return getSuccessResponse(regionService.searchRegionsByCode(regionCodes));
  }

  /** Ищет один регион по коду (slug) */
  @GetMapping(value = "/{regionCode}")
  @Operation(
      operationId = "searchRegionByCode",
      summary = "Предоставление информации о регионе по коду региона"
  )
  @ApiResponses({
      @ApiResponse(responseCode = "200", description = "Успешное получение информации о регионе"),
      @ApiResponse(responseCode = "400", description = "Некорректный код региона (валидация параметра)"),
      @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
  })
  public ResultObj<List<RegionDto>> searchRegionByCode(
      @PathVariable("regionCode")
      @NotBlank(message = "regionCode не должен быть пустым")
      String regionCode
  ) {
    return responseHandler.executeOrThrow(() ->
        getSuccessResponse(regionService.searchRegionsByCode(List.of(regionCode)))
    );
  }

  /** Возвращает полный список регионов (ТОЛЬКО явная ручка) */
  @GetMapping(value = "/all")
  @Operation(operationId = "getAllRegions", summary = "Получение полного перечня регионов")
  @ApiResponses({
      @ApiResponse(responseCode = "200", description = "Успешное получение списка регионов"),
      @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
  })
  public ResultObj<List<RegionDto>> getAllRegions() {
    return getSuccessResponse(regionService.getAllRegions());
  }
}

@Slf4j
@Service
@RequiredArgsConstructor
public class RegionService {

  public static final String REGION_BY_CODE = "region_by_code";

  private final CacheGetOrLoadService cacheGetOrLoadService;
  private final RegionCacheOps regionCacheOps;

  @Nonnull
  public List<RegionDto> searchRegionsByCode(@Nonnull List<String> regionCodes) {
    return cacheGetOrLoadService.fetchData(REGION_BY_CODE, regionCodes);
  }

  @Nonnull
  public List<RegionDto> getAllRegions() {
    return regionCacheOps.loadAllRegions();
  }
}

@Service
@RequiredArgsConstructor
public class RegionCacheOps {

  public static final String REGION_ALL = "region_all";

  private final BaseMasterDataRequestService baseMasterDataRequestService;
  private final SearchRequestProperties properties;
  private final RegionMapper regionMapper;

  /** Загружает и кеширует полный список регионов из АС Мастер-данные */
  @Cacheable(cacheNames = REGION_ALL, key = "'ALL'", sync = true)
  @Nonnull
  public List<RegionDto> loadAllRegions() {
    GetItemsSearchResponse response = baseMasterDataRequestService.requestData(
        properties.getSlugValueForRegion(),
        null,
        SearchRequestProperties.Context.BOOK
    );

    return createResult(response, regionMapper::mapItemToDto);
  }
}

@Component
@RequiredArgsConstructor
public class LoaderRegionByCode implements BatchLoader<RegionDto> {

  private final BaseMasterDataRequestService baseMasterDataRequestService;
  private final SearchRequestProperties properties;
  private final RegionMapper regionMapper;

  @Override
  public String cacheName() {
    return RegionService.REGION_BY_CODE;
  }

  @Override
  public Class<RegionDto> elementType() {
    return RegionDto.class;
  }

  @Override
  public String extractKey(RegionDto value) {
    return value.getRegionCode();
  }

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

    return createResult(response, regionMapper::mapItemToDto);
  }
}

@Component
public class RegionMapper {

  private static final String ID = "id";
  private static final String SLUG = "slug";
  private static final String NAME = "name";
  private static final String DESCRIPTION = "description";

  public RegionDto mapItemToDto(Map<String, String> item) {
    if (item == null || item.isEmpty()) {
      return null;
    }

    return RegionDto.builder()
        .uuid(item.get(ID))
        .regionCode(item.get(SLUG))
        .regionName(item.get(NAME))
        .regionDescription(item.get(DESCRIPTION))
        .build();
  }
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonIgnoreProperties(ignoreUnknown = true)
@Schema(description = "Информация о регионе")
public class RegionDto implements Serializable {

  @Schema(description = "UUID")
  private String uuid;

  @Schema(description = "Код региона")
  private String regionCode;

  @Schema(description = "Наименование региона")
  private String regionName;

  @Schema(description = "Описание региона")
  private String regionDescription;
}

slugValueForRegion: ${MASTER_DATA_SLUG_VALUE_FOR_REGION:RusFedSubjCodeFTS}
```
