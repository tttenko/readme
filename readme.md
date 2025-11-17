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

@Slf4j
@Service
@RequiredArgsConstructor
public class NdsService2 {

    /** Имя кеша для ставок НДС. */
    public static final String CACHE_NDS_BY_RATE = "nds_by_rate";

    /** Специальный ключ "все ставки". */
    public static final String ALL_KEY = "__ALL__";

    /** Формат даты, которая зашивается в ключ кеша. */
    public static final DateTimeFormatter DATE_TIME_FORMATTER =
            DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mmXXX");

    /** Тип ставки (базовая). Используется лоадером при запросе в МД. */
    public static final List<String> BASIC_RATE_TYPE = List.of("1");

    /** Признак активности ставки. Используется лоадером. */
    public static final List<String> ACTIVE_RATE = List.of("true");

    private final CacheGetOrLoadService cacheGetOrLoadService;

    /**
     * Возвращает базовые ставки НДС, применяя фильтры по дате, коду и ставке.
     */
    @NonNull
    public ResultObj<List<NdsDto>> getBasicVatRate(
            @Nullable final ZonedDateTime date,
            @Nullable final List<String> code,
            @Nullable final List<String> rate
    ) {
        // 1. Нормализуем дату (если не передана – берём сейчас)
        final ZonedDateTime targetDate = (date != null) ? date : ZonedDateTime.now();

        // 2. Формируем ключи кэша вида "<date>|<rate>" или просто "<date>" для "все ставки"
        final List<String> keys = buildKeys(targetDate, rate);

        // 3. Забираем бакеты из кеша/лоадера
        final List<RateBucket> buckets =
                cacheGetOrLoadService.fetchData(CACHE_NDS_BY_RATE, keys);

        // 4. Разворачиваем бакеты в список DTO с учётом фильтров
        final List<NdsDto> dto =
                mapBucketsToDtoList(buckets, targetDate, rate, code);

        // 5. Оборачиваем в ResultObj (как и раньше)
        return getSuccessResponse(dto);
    }

    /**
     * Ключ кэширования: "<date>|<rate>" или просто "<date>" (что означает "все ставки").
     */
    @NonNull
    private List<String> buildKeys(
            @NonNull ZonedDateTime date,
            @Nullable List<String> rate
    ) {
        final String datePart = DATE_TIME_FORMATTER.format(date);

        // Если ставки не переданы – один ключ с датой, внутри лоадер вернёт все ставки.
        if (rate == null || rate.isEmpty()) {
            return List.of(datePart);
        }

        final List<String> keys = rate.stream()
                .filter(Objects::nonNull)
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .map(r -> datePart + "|" + r)
                .toList();

        // Если после фильтрации ничего не осталось – тоже считаем, что нужно "всё".
        return keys.isEmpty() ? List.of(datePart) : keys;
    }

    /**
     * "Бакет" – ключ кеша + набор полных DTO, загруженных лоадером.
     * Лоадер `LoaderNdsByRate` возвращает именно такие объекты.
     */
    public record RateBucket(String key, Set<NdsFullDto> items) { }

    // === Преобразование бакетов в DTO ===

    @NonNull
    private List<NdsDto> mapBucketsToDtoList(
            @NonNull List<RateBucket> buckets,
            @NonNull ZonedDateTime date,          // сейчас фактически не используется, но оставлен на будущее
            @Nullable List<String> rate,
            @Nullable List<String> code
    ) {
        if (buckets.isEmpty()) {
            return List.of();
        }

        return buckets.stream()
                .flatMap(b -> b.items().stream())
                .filter(matches(rate, code))
                .map(this::toDto)
                .toList();
    }

    /**
     * Фильтрация по ставкам и кодам.
     * Фильтрацию по дате мы больше не делаем здесь – она уже учтена в запросе,
     * который формирует `LoaderNdsByRate`.
     */
    @NonNull
    private Predicate<NdsFullDto> matches(
            @Nullable List<String> rate,
            @Nullable List<String> code
    ) {
        final boolean hasRate = rate != null && !rate.isEmpty();
        final boolean hasCode = code != null && !code.isEmpty();

        final Set<String> rateSet = hasRate ? new HashSet<>(rate) : Set.of();
        final Set<String> codeSet = hasCode ? new HashSet<>(code) : Set.of();

        return e -> (!hasRate || rateSet.contains(e.getRate()))
                 && (!hasCode || codeSet.contains(e.getCode()));
    }

    @NonNull
    private NdsDto toDto(@NonNull NdsFullDto e) {
        return NdsDto.builder()
                .name(e.getName())
                .rate(e.getRate())
                .id(e.getId())
                .code(e.getCode())
                .build();
    }

    /**
     * Полностью очищает кеш ставок НДС.
     * Логика остаётся прежней – Spring-Cache сбрасывает кеш, а следующее обращение
     * снова пойдёт в `LoaderNdsByRate`.
     */
    @CacheEvict(cacheNames = CACHE_NDS_BY_RATE, allEntries = true)
    public void cleanCache() {
        log.info("NDS cache cleared");
    }

    // getSuccessResponse(...) – как и раньше, берёшь из базового класса / утилиты
}

```
