```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonIgnoreProperties(ignoreUnknown = true)
@Schema(description = "Информация о дате из производственного календаря")
public class ProdCalendDateDto implements Serializable {

    @Schema(description = "UUID записи")
    private String id;

    @Schema(description = "Дата из МД (например: YYYY-MM-DD 00:00:00.0)")
    private String date;

    @Schema(description = "Короткая дата (dd.MM.yyyy)")
    private String dateShort;

    @Schema(description = "Код типа дня (например: 1 - рабочий, 4 - нерабочий)")
    private String dateType;

    @Schema(description = "Описание типа дня")
    private String dayTypeDescription;
}

@Component
@RequiredArgsConstructor
public class LoaderProdCalendDateByDate implements BatchLoader<ProdCalendDateDto> {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final ProdCalendDateMapper prodCalendDateMapper;

    @Override
    public String cacheName() {
        return ProdCalendDateService.PROD_CALEND_DATE_BY_DATE;
    }

    @Override
    public Class<ProdCalendDateDto> elementType() {
        return ProdCalendDateDto.class;
    }

    /**
     * Ключ кеша должен совпадать с входным ключом (dd.MM.yyyy).
     * Мы кладём в dto dateShort (dd.MM.yyyy), поэтому его и используем.
     */
    @Override
    public String extractKey(ProdCalendDateDto value) {
        return value.getDateShort();
    }

    @Override
    public List<ProdCalendDateDto> fetchByKeys(List<String> keys) {
        if (keys == null || keys.isEmpty()) {
            return List.of();
        }

        var response = baseMasterDataRequestService.requestDataWithAttribute(
            properties.getSlugValueForProdCalendDate(), // RussianPCDate
            keys,
            SearchRequestProperties.Context.BOOK
        );

        return BaseMasterDataRequestService.createResultWithAttribute(response, prodCalendDateMapper);
    }
}

@Service
@RequiredArgsConstructor
public class ProdCalendDateService {

    public static final String PROD_CALEND_DATE_BY_DATE = "prod_calend_date_by_date";

    private static final ZoneId MOSCOW = ZoneId.of("Europe/Moscow");
    private static final DateTimeFormatter INPUT_FMT = DateTimeFormatter.ofPattern("dd.MM.yyyy");

    private final CacheGetOrLoadService cacheGetOrLoadService;

    public List<ProdCalendDateDto> searchByDates(List<String> dates) {
        List<String> keys = normalizeDates(dates);
        return cacheGetOrLoadService.fetchData(PROD_CALEND_DATE_BY_DATE, keys);
    }

    private List<String> normalizeDates(List<String> dates) {
        if (dates == null || dates.isEmpty()) {
            return List.of(LocalDate.now(MOSCOW).format(INPUT_FMT));
        }

        List<String> cleaned = dates.stream()
            .filter(Objects::nonNull)
            .map(String::trim)
            .filter(s -> !s.isEmpty())
            .distinct()
            .toList();

        return cleaned.isEmpty()
            ? List.of(LocalDate.now(MOSCOW).format(INPUT_FMT))
            : cleaned;
    }
}

@RestController
@Validated
@RequiredArgsConstructor
public class ProdCalendDateControllerImpl implements ProdCalendDateController {

    private final ProdCalendDateService prodCalendDateService;
    private final ResponseHandler responseHandler;

    @Override
    public ResultObj<List<ProdCalendDateDto>> searchProdCalendDates(List<String> dates) {
        // как у country: без executeOrThrow для bulk
        return BaseMasterDataRequestService.getSuccessResponse(
            prodCalendDateService.searchByDates(dates)
        );
    }

    @Override
    public ResultObj<List<ProdCalendDateDto>> searchProdCalendDateByDate(String date) {
        // как у country: single через responseHandler
        return responseHandler.executeOrThrow().getSuccessResponse(
            prodCalendDateService.searchByDates(List.of(date))
        );
    }
}

@Slf4j
@Component
public class ProdCalendDateMapper implements DataMapperWithAttribute<ProdCalendDateDto> {

    private static final String SLUG_DATE = "date";
    private static final String SLUG_DATE_TYPE = "dateType";

    private static final String VALUE_REF = "valueRef";
    private static final String VALUE_REF_ITEM = "valueRefItem";
    private static final String ITEM = "item";
    private static final String SLUG = "slug";
    private static final String NAME = "name";

    private static final DateTimeFormatter SHORT_FMT = DateTimeFormatter.ofPattern("dd.MM.yyyy");

    @Override
    public ProdCalendDateDto mapValuesToDto(
        Map<String, Object> values,
        Map<String, Map<String, Object>> attributes
    ) {
        ProdCalendDateDto dto = new ProdCalendDateDto();

        dto.setId(getValueOrNull(values, ITEM_ID));

        String fullDate = getAttributeValue(attributes, SLUG_DATE);
        dto.setDate(fullDate);
        dto.setDateShort(toShortDate(fullDate));

        Map<String, Object> dateTypeNode = attributes == null ? null : attributes.get(SLUG_DATE_TYPE);
        dto.setDateType(extractDateTypeSlug(dateTypeNode));
        dto.setDayTypeDescription(extractDateTypeName(dateTypeNode));

        return dto;
    }

    private String toShortDate(String fullDate) {
        if (fullDate == null || fullDate.length() < 10) {
            return null;
        }
        try {
            // "2026-01-01 00:00:00.0" -> берём первые 10 символов ISO-даты
            LocalDate ld = LocalDate.parse(fullDate.substring(0, 10));
            return ld.format(SHORT_FMT);
        } catch (Exception e) {
            log.warn("Cannot parse date from MD value: {}", fullDate, e);
            return null;
        }
    }

    private String extractDateTypeSlug(Map<String, Object> dateTypeNode) {
        if (dateTypeNode == null) return null;

        var valueRefItem = Mono.of(dateTypeNode)
            .extract(VALUE_REF)
            .extract(VALUE_REF_ITEM);

        // вариант 1: valueRefItem.slug
        var directSlug = valueRefItem.extract(SLUG);
        if (!directSlug.isNothing()) {
            return directSlug.getString();
        }

        // вариант 2 (как в вашем примере): valueRefItem.item.slug
        return valueRefItem.extract(ITEM).extract(SLUG).getString();
    }

    private String extractDateTypeName(Map<String, Object> dateTypeNode) {
        if (dateTypeNode == null) return null;

        var valueRefItem = Mono.of(dateTypeNode)
            .extract(VALUE_REF)
            .extract(VALUE_REF_ITEM);

        var directName = valueRefItem.extract(NAME);
        if (!directName.isNothing()) {
            return directName.getString();
        }

        return valueRefItem.extract(ITEM).extract(NAME).getString();
    }
}
```
