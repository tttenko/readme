```java

@Component
@RequiredArgsConstructor
public class LoaderNdsByRate implements BatchLoader<NdsService2.RateBucket> {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final NdsMapper ndsMapper;
    private final SearchRequestProperties properties;

    @Override
    public String cacheName() {
        return NdsService2.CACHE_NDS_BY_RATE;
    }

    @Override
    public Class<NdsService2.RateBucket> elementType() {
        return NdsService2.RateBucket.class;
    }

    @Override
    public String extractKey(NdsService2.RateBucket value) {
        return value.key();
    }

    @Override
    @NonNull
    public List<NdsService2.RateBucket> fetchByKeys(@NonNull List<String> keys) {
        if (keys.isEmpty()) {
            return List.of();
        }

        // 1. дата вшита в ключ, берём её из первого ключа
        String dateString = extractDate(keys.get(0));

        // 2. грузим все записи НДС на эту дату
        List<NdsFullDto> all = loadAllFromMd(dateString);

        // 3. группируем по ставке
        Map<String, Set<NdsFullDto>> byRate = groupByRate(all);
        Set<NdsFullDto> union = Set.copyOf(all);

        // 4. собираем бакеты под каждый ключ
        return buildBuckets(keys, byRate, union);
    }

    // === ВСПОМОГАТЕЛЬНЫЕ МЕТОДЫ РАЗБОРА КЛЮЧА ===

    /** Возвращает часть до разделителя: "<date>|<rate>" -> "<date>". */
    private static String extractDate(@NonNull String key) {
        int idx = key.indexOf('|');
        return (idx < 0) ? key : key.substring(0, idx);
    }

    /** Возвращает часть после разделителя или ALL_KEY, если её нет. */
    private static String extractRateKey(@NonNull String key) {
        int idx = key.indexOf('|');
        if (idx < 0 || idx == key.length() - 1) {
            return NdsService2.ALL_KEY;
        }
        return key.substring(idx + 1);
    }

    // === ВСПОМОГАТЕЛЬНАЯ ЛОГИКА ГРУППИРОВКИ И СБОРКИ БАКЕТОВ ===

    /** Группирует все записи по ставке. */
    private Map<String, Set<NdsFullDto>> groupByRate(@NonNull List<NdsFullDto> all) {
        return all.stream()
                .collect(Collectors.groupingBy(
                        NdsFullDto::getRate,
                        LinkedHashMap::new,
                        Collectors.toCollection(LinkedHashSet::new)
                ));
    }

    /** Собирает список бакетов под каждый запрошенный ключ. */
    private List<NdsService2.RateBucket> buildBuckets(@NonNull List<String> keys,
                                                      @NonNull Map<String, Set<NdsFullDto>> byRate,
                                                      @NonNull Set<NdsFullDto> union) {
        return keys.stream()
                .map(rawKey -> {
                    String rateKey = extractRateKey(rawKey);
                    Set<NdsFullDto> items = resolveItemsForRateKey(rateKey, byRate, union);
                    return new NdsService2.RateBucket(rawKey, Set.copyOf(items));
                })
                .toList();
    }

    /** Возвращает набор элементов для конкретного rate-ключа. */
    private Set<NdsFullDto> resolveItemsForRateKey(@NonNull String rateKey,
                                                   @NonNull Map<String, Set<NdsFullDto>> byRate,
                                                   @NonNull Set<NdsFullDto> union) {
        return NdsService2.ALL_KEY.equals(rateKey)
                ? union
                : byRate.getOrDefault(rateKey, Set.of());
    }

    // === ЗАГРУЗКА ИЗ МД ===

    @NonNull
    private List<NdsFullDto> loadAllFromMd(@NonNull String dateString) {
        GetItemsSearchResponse response = baseMasterDataRequestService.requestData(
                buildRequest(dateString),
                SearchRequestProperties.Context.BOOK
        );
        return BaseMasterDataRequestService.createResultWithAttribute(response, ndsMapper);
    }

    @NonNull
    private ItemsSearchCriteriaRequest buildRequest(@NonNull String dateString) {
        return RequestFactory.getByAttrValuesBuilder()
                .dictionaryName(properties.getSlugValueForVat())
                .addAttributesAndItemSlug(
                        properties.getAttributeIdForTaxRateType(),
                        NdsService2.BASIC_RATE_TYPE
                )
                .addAttributesAndValue(
                        properties.getAttributeIdForTaxRateActive(),
                        Pair.of(NdsService2.ACTIVE_RATE, RequestFactory.BOOLEAN_OPERATION)
                )
                .addAttributesAndValue(
                        properties.getAttributeIdForTaxRateEndDate(),
                        Pair.of(List.of(dateString), RequestFactory.MORE_OPERATION)
                )
                .addAttributesAndValue(
                        properties.getAttributeIdForTaxRateStartDate(),
                        Pair.of(List.of(dateString), RequestFactory.LESS_OPERATION)
                )
                .build();
    }
}

@NonNull
    private ItemsSearchCriteriaRequest buildRequest(@NonNull String dateString) {
        return RequestFactory.getByAttrValuesBuilder()
                .dictionaryName(properties.getSlugValueForVat())
                .addAttributesAndRefItemSlug(
                        properties.getAttributeIdForTaxRateType(),
                        NdsService2.BASIC_RATE_TYPE
                )
                .addAttributesAndValue(
                        properties.getAttributeIdForTaxRateActive(),
                        Pair.of(NdsService2.ACTIVE_RATE, RequestFactory.BOOLEAN_OPERATION)
                )
                .addAttributesAndValue(
                        properties.getAttributeIdForTaxRateEndDate(),
                        Pair.of(List.of(dateString), RequestFactory.MORE_OPERATION)
                )
                .addAttributesAndValue(
                        properties.getAttributeIdForTaxRateStartDate(),
                        Pair.of(List.of(dateString), RequestFactory.LESS_OPERATION)
                )
                .build();
    }

```
