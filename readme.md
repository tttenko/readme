```java
@Slf4j
@Service
@RequiredArgsConstructor
public class NdsService2 {

    public static final String CACHE_NDS_BY_RATE = "nds_by_rate";
    private static final String ALL_KEY = "_ALL_";
    private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mmXXX");

    private static final List<String> BASIC_RATE_TYPE = List.of("1");
    private static final List<String> ACTIVE_RATE = List.of("true");

    private final NdsMapper ndsMapper;
    private final BatchCacheSupport batchLoad;
    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;

    private record RateBucket(@Nonnull String key, @Nonnull Set<NdsFullDto> items) {}

    @Nonnull
    public ResultDto<List<NdsDto>> getBasicVatRate(@Nullable final ZonedDateTime date,
                                                   @Nullable final List<String> code,
                                                   @Nullable final List<String> rate) {
        final ZonedDateTime targetDate = (date != null) ? date : ZonedDateTime.now();

        // какие ключи нужны из кэша?
        final List<String> keys = CollectionUtils.isEmpty(rate) ? List.of(ALL_KEY) : new ArrayList<>(rate);

        // батч-чтение/дозагрузка промахов (один вызов загрузчика на весь miss)
        final List<RateBucket> buckets = batchLoad.fetchBatch(
                CACHE_NDS_BY_RATE,
                keys,
                miss -> loadAllFromMdAndPartition(targetDate, miss),
                RateBucket::key,
                RateBucket.class
        );

        // stream → filters → map → collect
        final var dto = toDto(
                applyFilters(
                        asStream(buckets),
                        targetDate, rate, code
                )
        ).toList();

        // используем готовый статический хелпер проекта
        return BaseMasterDataRequestService.getSuccessResponse(dto);
    }

    @Nonnull
    private Stream<NdsFullDto> asStream(@Nonnull final List<RateBucket> buckets) {
        return buckets.stream()
                .filter(Objects::nonNull)
                .flatMap(b -> b.items().stream());
    }

    @Nonnull
    private Stream<NdsFullDto> applyFilters(@Nonnull final Stream<NdsFullDto> source,
                                            @Nonnull final ZonedDateTime date,
                                            @Nullable final List<String> rate,
                                            @Nullable final List<String> code) {
        Stream<NdsFullDto> s = source
                .filter(e -> isBefore(date, e) && isAfter(date, e));
        if (!CollectionUtils.isEmpty(rate)) s = s.filter(e -> rate.contains(e.getRate()));
        if (!CollectionUtils.isEmpty(code)) s = s.filter(e -> code.contains(e.getCode()));
        return s;
    }

    @Nonnull
    private Stream<NdsDto> toDto(@Nonnull final Stream<NdsFullDto> source) {
        return source.map(e -> NdsDto.builder()
                .name(e.getName())
                .rate(e.getRate())
                .id(e.getId())
                .code(e.getCode())
                .name(e.getName())
                .build());
    }

    @Nonnull
    private List<RateBucket> loadAllFromMdAndPartition(@Nonnull final ZonedDateTime date,
                                                       @Nonnull final List<String> missKeys) {
        final String dateString = date.format(DATE_TIME_FORMATTER);

        final var response = baseMasterDataRequestService.requestData(
                buildRequest(dateString),
                SearchRequestProperties.Context.BOOK
        );

        final List<NdsFullDto> all = createResultWithAttribute(response, ndsMapper);

        // группировка по rate
        final Map<String, Set<NdsFullDto>> byRate = new HashMap<>();
        for (NdsFullDto e : all) {
            byRate.computeIfAbsent(e.getRate(), r -> new CopyOnWriteArraySet<>()).add(e);
        }

        final List<RateBucket> out = new ArrayList<>(missKeys.size());
        for (String miss : missKeys) {
            if (ALL_KEY.equals(miss)) {
                out.add(new RateBucket(ALL_KEY, new CopyOnWriteArraySet<>(all)));
            } else {
                out.add(new RateBucket(miss, new CopyOnWriteArraySet<>(byRate.getOrDefault(miss, Set.of()))));
            }
        }
        return out;
    }

    @Nonnull
    private ItemsSearchCriteriaRequest buildRequest(@Nonnull final String dateString) {
        return RequestFactory.getByAttrValuesBuilder()
                .dictionaryName(properties.getSlugValueForVat())
                .addAttributesAndRefItemSlug(properties.getAttributeIdForTaxRateType(), BASIC_RATE_TYPE)
                .addAttributesAndValue(
                        properties.getAttributeIdForTaxRateActive(),
                        Pair.of(ACTIVE_RATE, RequestFactory.BOOLEAN_OPERATION))
                .addAttributesAndValue(
                        properties.getAttributeIdForTaxRateEndDate(),
                        Pair.of(List.of(dateString), RequestFactory.MORE_OPERATION))
                .addAttributesAndValue(
                        properties.getAttributeIdForTaxRateStartDate(),
                        Pair.of(List.of(dateString), RequestFactory.LESS_OPERATION))
                .build();
    }

    private static boolean isAfter(@Nonnull final ZonedDateTime date, @Nonnull final NdsFullDto e) {
        return e.getRateDateEndZoned() == null || e.getRateDateEndZoned().isAfter(date);
    }

    private static boolean isBefore(@Nonnull final ZonedDateTime date, @Nonnull final NdsFullDto e) {
        return e.getRateDateStartZoned() != null && e.getRateDateStartZoned().isBefore(date);
    }

    @CacheEvict(cacheNames = CACHE_NDS_BY_RATE, allEntries = true)
    public void cleanCache() {
        log.info("NDS cache cleared");
    }
}

------------------------------------------------------------2--------------------------------------------------------------

@Slf4j
@Service
@RequiredArgsConstructor
public class NdsService2 {

  public static final String CACHE_NDS_BY_RATE = "nds_by_rate";
  private static final String ALL_KEY = "__ALL__";
  private static final DateTimeFormatter DATE_TIME_FORMATTER =
      DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mmXXX");

  private static final List<String> BASIC_RATE_TYPE = List.of("1");
  private static final List<String> ACTIVE_RATE = List.of("true");

  private final NdsMapper ndsMapper;
  private final BatchCacheSupport batchLoad;
  private final BaseMasterDataRequestService baseMasterDataRequestService;
  private final SearchRequestProperties properties;

  /** Носитель порции данных в кеше (один ключ — один набор элементов). */
  private record RateBucket(@Nonnull String key, @Nonnull Set<NdsFullDto> items) {}

  // ---------------------- API ----------------------

  @Nonnull
  public ResultObj<List<NdsDto>> getBasicVatRate(@Nullable final ZonedDateTime date,
                                                 @Nullable final List<String> code,
                                                 @Nullable final List<String> rate) {
    final ZonedDateTime targetDate = (date != null) ? date : ZonedDateTime.now();

    // какие ключи нужны из кеша
    final List<String> keys = (rate == null || rate.isEmpty())
        ? List.of(ALL_KEY)
        : List.copyOf(rate);

    // батч-чтение/догрузка miss (один вызов загрузчика на весь miss)
    final List<RateBucket> buckets = batchLoad.fetchBatch(
        CACHE_NDS_BY_RATE,
        keys,
        miss -> partitionByRate(loadAllFromMd(targetDate), miss),
        RateBucket::key,
        RateBucket.class
    );

    final List<NdsDto> dto = buckets.stream()
        .flatMap(b -> b.items().stream())
        .filter(matches(targetDate, rate, code))
        .map(this::toDto)
        .toList();

    return BaseMasterDataRequestService.getSuccessResponse(dto);
  }

  // -------------------- Загрузка и разбиение --------------------

  /** Разовый запрос в МД без фильтра по rate (как и в старой логике). */
  @Nonnull
  private List<NdsFullDto> loadAllFromMd(@Nonnull final ZonedDateTime date) {
    final String dateString = date.format(DATE_TIME_FORMATTER);
    final var response = baseMasterDataRequestService.requestData(
        buildRequest(dateString),
        Context.BOOK
    );
    return BaseMasterDataRequestService.createResultWithAttribute(response, ndsMapper);
  }

  /** Группировка по rate и формирование бакетов для всех miss-ключей. */
  @Nonnull
  private List<RateBucket> partitionByRate(@Nonnull final List<NdsFullDto> all,
                                           @Nonnull final List<String> missKeys) {
    final Map<String, Set<NdsFullDto>> byRate = all.stream()
        .collect(Collectors.groupingBy(
            NdsFullDto::getRate,
            Collectors.toCollection(LinkedHashSet::new)));

    final Set<NdsFullDto> union = Set.copyOf(all);

    return missKeys.stream()
        .map(k -> new RateBucket(
            k,
            Set.copyOf(ALL_KEY.equals(k) ? union : byRate.getOrDefault(k, Set.of()))
        ))
        .toList();
  }

  // -------------------- Фильтрация и маппинг --------------------

  @Nonnull
  private Predicate<NdsFullDto> matches(@Nonnull final ZonedDateTime date,
                                        @Nullable final List<String> rate,
                                        @Nullable final List<String> code) {
    final boolean hasRate = rate != null && !rate.isEmpty();
    final boolean hasCode = code != null && !code.isEmpty();
    return e -> isBefore(date, e) && isAfter(date, e)
        && (!hasRate || rate.contains(e.getRate()))
        && (!hasCode || code.contains(e.getCode()));
  }

  @Nonnull
  private NdsDto toDto(@Nonnull final NdsFullDto e) {
    return NdsDto.builder()
        .name(e.getName())
        .rate(e.getRate())
        .id(e.getId())
        .code(e.getCode())
        .build();
  }

  // -------------------- Построение запроса в МД --------------------

  @Nonnull
  private ItemsSearchCriteriaRequest buildRequest(@Nonnull final String dateString) {
    return RequestFactory.getByAttrValuesBuilder()
        .dictionaryName(properties.getSlugValueForVat())
        .addAttributesAndRefItemSlug(properties.getAttributeIdForTaxRateType(), BASIC_RATE_TYPE)
        .addAttributesAndValue(
            properties.getAttributeIdForTaxRateActive(),
            Pair.of(ACTIVE_RATE, RequestFactory.BOOLEAN_OPERATION))
        .addAttributesAndValue(
            properties.getAttributeIdForTaxRateEndDate(),
            Pair.of(List.of(dateString), RequestFactory.MORE_OPERATION))
        .addAttributesAndValue(
            properties.getAttributeIdForTaxRateStartDate(),
            Pair.of(List.of(dateString), RequestFactory.LESS_OPERATION))
        .build();
  }

  // -------------------- Вспомогательные условия --------------------

  private static boolean isAfter(@Nonnull final ZonedDateTime date, @Nonnull final NdsFullDto e) {
    return e.getRateDateEndZoned() == null || e.getRateDateEndZoned().isAfter(date);
  }

  private static boolean isBefore(@Nonnull final ZonedDateTime date, @Nonnull final NdsFullDto e) {
    return e.getRateDateStartZoned() != null && e.getRateDateStartZoned().isBefore(date);
  }

  // -------------------- Инвалидация кеша --------------------

  @CacheEvict(cacheNames = CACHE_NDS_BY_RATE, allEntries = true)
  public void cleanCache() {
    log.info("NDS cache cleared");
  }
}

```
