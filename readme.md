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
    public ResultObj<List<NdsDto>> getBasicVatRate(@Nullable final ZonedDateTime date,
                                                   @Nullable final List<String> code,
                                                   @Nullable final List<String> rate) {
        final ZonedDateTime targetDate = (date != null) ? date : ZonedDateTime.now();

        final List<String> keys = (rate == null || rate.isEmpty())
                ? List.of(ALL_KEY)
                : List.copyOf(rate);

        final List<RateBucket> buckets = batchLoad.fetchBatch(
                CACHE_NDS_BY_RATE,
                keys,
                miss -> partitionByRate(loadAllFromMd(targetDate), miss),
                RateBucket::key,
                RateBucket.class
        );

        final List<NdsDto> dto = mapBucketsToDtoList(buckets, targetDate, rate, code);

        return getSuccessResponse(dto);
    }

    @Nonnull
    private List<NdsFullDto> loadAllFromMd(@Nonnull final ZonedDateTime date) {
        final String dateString = date.format(DATE_TIME_FORMATTER);
        final var response = baseMasterDataRequestService.requestData(
                buildRequest(dateString),
                SearchRequestProperties.Context.BOOK
        );
        return createResultWithAttribute(response, ndsMapper);
    }

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

    @Nonnull
    private List<NdsDto> mapBucketsToDtoList(@Nonnull final List<RateBucket> buckets,
                                             @Nonnull final ZonedDateTime dateTime,
                                             @Nullable final List<String> rate,
                                             @Nullable final List<String> code) {
        return buckets.stream()
                .flatMap(b -> b.items().stream())
                .filter(matches(dateTime, rate, code))
                .map(this::toDto)
                .toList();
    }

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

---------------------------------------------------------------Тесты---------------------------------------------------------------------------

/** Лёгкие ассершены для проверок кеша nds_by_rate в интеграционных тестах. */
public final class CacheAssertions {

  private CacheAssertions() {}

  /** Проверяет, что в кеше ровно эти ключи (порядок не важен). */
  public static void assertCacheHasKeys(CacheManager mgr, String cacheName, String... expectedKeys) {
    assertThat(CacheIntrospection.keys(mgr, cacheName))
        .containsExactlyInAnyOrder(expectedKeys);
  }

  /** Проверяет, что кеш пуст. */
  public static void assertCacheEmpty(CacheManager mgr, String cacheName) {
    assertThat(CacheIntrospection.keys(mgr, cacheName)).isEmpty();
  }

  /** Возвращает типизированный бакет по ключу. */
  public static RateBucket bucket(CacheManager mgr, String cacheName, String key) {
    Object raw = CacheIntrospection.rawValue(mgr, cacheName, key);
    assertThat(raw).as("Bucket '%s' должен существовать", key)
        .isInstanceOf(RateBucket.class);
    return (RateBucket) raw;
  }

  /** Проверяет размер набора элементов в бакете. */
  public static void assertBucketSize(CacheManager mgr, String cacheName, String key, int expected) {
    assertThat(bucket(mgr, cacheName, key).items()).hasSize(expected);
  }

  /** Проверяет, что бакет содержит элементы с указанными id. */
  public static void assertBucketContainsIds(CacheManager mgr, String cacheName, String key, String... ids) {
    Set<NdsFullDto> items = bucket(mgr, cacheName, key).items();
    assertThat(items).extracting(NdsFullDto::getId)
        .containsExactlyInAnyOrder(ids);
  }
}

package ru.sber.cs.supplier.portal.masterdata.services.impl.nds;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;
import static ru.sber.cs.supplier.portal.masterdata.services.impl.nds.NdsService2.CACHE_NDS_BY_RATE;
import static ru.sber.cs.supplier.portal.masterdata.services.support.CacheAssertions.assertBucketContainsIds;
import static ru.sber.cs.supplier.portal.masterdata.services.support.CacheAssertions.assertBucketSize;
import static ru.sber.cs.supplier.portal.masterdata.services.support.CacheAssertions.assertCacheEmpty;
import static ru.sber.cs.supplier.portal.masterdata.services.support.CacheAssertions.assertCacheHasKeys;

import com.github.benmanes.caffeine.cache.Caffeine;
import java.time.Duration;
import java.time.ZonedDateTime;
import java.util.List;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.mockito.MockedStatic;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import ru.sber.cs.supplier.portal.masterdata.models.adapter.dto.NdsDto;
import ru.sber.cs.supplier.portal.masterdata.models.adapter.dto.NdsFullDto;
import ru.sber.cs.supplier.portal.masterdata.request.ItemsSearchCriteriaRequest;
import ru.sber.cs.supplier.portal.masterdata.services.BaseMasterDataRequestService;
import ru.sber.cs.supplier.portal.masterdata.services.GetItemsSearchResponse;
import ru.sber.cs.supplier.portal.masterdata.services.impl.support.BatchCacheSupport;
import ru.sber.cs.supplier.portal.masterdata.services.mapper.NdsMapper;
import ru.sber.cs.supplier.portal.masterdata.settings.SearchRequestProperties;
import ru.sber.cs.supplier.portal.masterdata.settings.SearchRequestProperties.Context;

@SpringBootTest(classes = {
    NdsService2CacheTest.TestCacheConfig.class,
    NdsService2.class
})
class NdsService2CacheTest {

  @TestConfiguration
  @EnableCaching
  static class TestCacheConfig {
    @Bean
    Caffeine<Object, Object> caffeine() {
      return Caffeine.newBuilder()
          .maximumSize(100)
          .expireAfterWrite(Duration.ofMinutes(10))
          .recordStats();
    }

    @Bean
    CacheManager cacheManager(final Caffeine<Object, Object> caffeine) {
      final var mgr = new CaffeineCacheManager(CACHE_NDS_BY_RATE);
      mgr.setCaffeine(caffeine);
      return mgr;
    }

    @Bean
    BatchCacheSupport batchCacheSupport(final CacheManager cm) {
      return new BatchCacheSupport(cm);
    }
  }

  @Autowired private NdsService2 service;
  @Autowired private CacheManager cacheManager;

  @MockBean private BaseMasterDataRequestService baseService;
  @MockBean private NdsMapper ndsMapper; // не используется напрямую, но нужен для конструктора
  @MockBean private SearchRequestProperties properties;

  private void stubProps() {
    when(properties.getSlugValueForVat()).thenReturn("vat");
    when(properties.getAttributeIdForTaxRateType()).thenReturn("type");
    when(properties.getAttributeIdForTaxRateActive()).thenReturn("active");
    when(properties.getAttributeIdForTaxRateEndDate()).thenReturn("end");
    when(properties.getAttributeIdForTaxRateStartDate()).thenReturn("start");
  }

  @Test
  @DisplayName("Warm-up → ключ появляется; повторный вызов — hit; cleanCache очищает")
  void warmup_hit_evict() {
    stubProps();
    final var date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");

    final var items = List.of(
        NdsFullDto.builder()
            .id("1").name("N1").rate("5").code("A")
            .rateDateStartZoned(date.minusDays(1))
            .rateDateEndZoned(null)
            .build()
    );

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {

      when(baseService.requestData(any(ItemsSearchCriteriaRequest.class), eq(Context.BOOK)))
          .thenReturn(new GetItemsSearchResponse());
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(items);

      // прогрев кеша
      service.getBasicVatRate(date, null, List.of("5"));
      assertCacheHasKeys(cacheManager, CACHE_NDS_BY_RATE, "5");
      assertBucketSize(cacheManager, CACHE_NDS_BY_RATE, "5", 1);
      verify(baseService, times(1)).requestData(any(), eq(Context.BOOK));

      // повторный вызов по тому же rate — hit
      service.getBasicVatRate(date, null, List.of("5"));
      verify(baseService, times(1)).requestData(any(), eq(Context.BOOK));

      // очистка кеша
      service.cleanCache();
      assertCacheEmpty(cacheManager, CACHE_NDS_BY_RATE);

      // снова прогрев → второй поход в МД
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(items);
      service.getBasicVatRate(date, null, List.of("5"));
      verify(baseService, times(2)).requestData(any(), eq(Context.BOOK));
    }
  }

  @Test
  @DisplayName("Без rate создаётся только бакет '__ALL__'")
  void no_rate_creates_all_bucket() {
    stubProps();
    final var date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");

    final var items = List.of(
        NdsFullDto.builder().id("1").name("N1").rate("5").code("A")
            .rateDateStartZoned(date.minusDays(5)).build(),
        NdsFullDto.builder().id("2").name("N2").rate("10").code("B")
            .rateDateStartZoned(date.minusDays(5)).build()
    );

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {

      when(baseService.requestData(any(ItemsSearchCriteriaRequest.class), eq(Context.BOOK)))
          .thenReturn(new GetItemsSearchResponse());
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(items);

      service.getBasicVatRate(date, null, null);

      assertCacheHasKeys(cacheManager, CACHE_NDS_BY_RATE, "__ALL__");
      assertBucketContainsIds(cacheManager, CACHE_NDS_BY_RATE, "__ALL__", "1", "2");
    }
  }

  @Test
  @DisplayName("Новый rate → добавляется новый бакет в кеше")
  void new_rate_adds_new_bucket() {
    stubProps();
    final var date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");

    final var items5 = List.of(
        NdsFullDto.builder().id("1").name("N1").rate("5").code("A")
            .rateDateStartZoned(date.minusDays(5)).build()
    );
    final var items10 = List.of(
        NdsFullDto.builder().id("2").name("N2").rate("10").code("B")
            .rateDateStartZoned(date.minusDays(5)).build()
    );

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {

      when(baseService.requestData(any(ItemsSearchCriteriaRequest.class), eq(Context.BOOK)))
          .thenReturn(new GetItemsSearchResponse());

      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(items5);
      service.getBasicVatRate(date, null, List.of("5"));
      assertCacheHasKeys(cacheManager, CACHE_NDS_BY_RATE, "5");

      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(items10);
      service.getBasicVatRate(date, null, List.of("10"));
      assertCacheHasKeys(cacheManager, CACHE_NDS_BY_RATE, "5", "10");
      assertBucketContainsIds(cacheManager, CACHE_NDS_BY_RATE, "10", "2");
    }
  }
}


@ExtendWith(MockitoExtension.class)
class NdsService2Test {

  @Mock private NdsMapper ndsMapper;
  @Mock private BatchCacheSupport batchLoad;
  @Mock private BaseMasterDataRequestService baseService;
  @Mock private SearchRequestProperties properties;

  @Captor private ArgumentCaptor<List<String>> keysCaptor;
  @Captor private ArgumentCaptor<ItemsSearchCriteriaRequest> reqCaptor;

  private NdsService2 service;

  @BeforeEach
  void setUp() {
    service = new NdsService2(ndsMapper, batchLoad, baseService, properties);

    // заглушки свойств, чтобы buildRequest собрал запрос
    when(properties.getSlugValueForVat()).thenReturn("vat");
    when(properties.getAttributeIdForTaxRateType()).thenReturn("type");
    when(properties.getAttributeIdForTaxRateActive()).thenReturn("active");
    when(properties.getAttributeIdForTaxRateEndDate()).thenReturn("end");
    when(properties.getAttributeIdForTaxRateStartDate()).thenReturn("start");
  }

  @Test
  @DisplayName("Если rate=null/empty → запрашиваем из кеша спец‑ключ __ALL__")
  void keys_whenRateNull_thenAllKey() {
    // эмулируем поведение BatchCacheSupport: отдадим пусто, нам важно только какие keys пришли
    doAnswer(inv -> {
      List<String> keys = inv.getArgument(1);
      return List.of(); // пустой список бакетов
    }).when(batchLoad).fetchBatch(eq(CACHE_NDS_BY_RATE), keysCaptor.capture(), any(), any(), any());

    try (MockedStatic<BaseMasterDataRequestService> statics = Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {
      when(baseService.requestData(any(ItemsSearchCriteriaRequest.class), eq(Context.BOOK)))
          .thenReturn(new GetItemsSearchResponse());

      ResultObj<List<NdsDto>> result = service.getBasicVatRate(ZonedDateTime.now(), null, null);

      assertThat(result).isNotNull();
      assertThat(keysCaptor.getValue()).containsExactly("__ALL__"); // ключи для кеша
    }
  }

  @Test
  @DisplayName("Если rate задан → ключи в батче совпадают со значениями rate")
  void keys_whenRateProvided_thenExactKeys() {
    var rates = List.of("5", "10");

    doAnswer(inv -> {
      List<String> keys = inv.getArgument(1);
      return List.of();
    }).when(batchLoad).fetchBatch(eq(CACHE_NDS_BY_RATE), keysCaptor.capture(), any(), any(), any());

    try (MockedStatic<BaseMasterDataRequestService> statics = Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {
      when(baseService.requestData(any(ItemsSearchCriteriaRequest.class), eq(Context.BOOK)))
          .thenReturn(new GetItemsSearchResponse());

      service.getBasicVatRate(ZonedDateTime.now(), null, rates);

      assertThat(keysCaptor.getValue()).containsExactlyElementsOf(rates);
    }
  }

  @Test
  @DisplayName("buildRequest: в requestData уходит словарь из properties и присутствует дата запроса")
  void buildRequest_shouldPassDictionaryAndDate() {
    // заставляем BatchCacheSupport вызвать наш loader (будем считать, что всё miss)
    doAnswer(inv -> {
      @SuppressWarnings("unchecked")
      Function<List<String>, List<Object>> loader = (Function<List<String>, List<Object>>) inv.getArgument(2);
      List<String> keys = inv.getArgument(1);
      return loader.apply(keys);
    }).when(batchLoad).fetchBatch(anyString(), anyList(), any(), any(), any());

    // статический маппинг ответа МД → список доменных объектов
    try (MockedStatic<BaseMasterDataRequestService> statics =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {

      // МД вернёт пустую обёртку, а маппер вернёт пустой список
      when(baseService.requestData(reqCaptor.capture(), eq(Context.BOOK)))
          .thenReturn(new GetItemsSearchResponse());
      statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(List.of());

      service.getBasicVatRate(ZonedDateTime.parse("2025-07-21T10:00:00+03:00"), null, null);

      ItemsSearchCriteriaRequest req = reqCaptor.getValue();
      assertThat(req).isNotNull();
      // ↓ используйте доступные геттеры вашего класса ItemsSearchCriteriaRequest
      assertThat(req.getDictionaryName()).isEqualTo("vat");
      assertThat(req.toString()).contains("2025-07-21"); // дата должна фигурировать среди условий
    }
  }

  @Test
  @DisplayName("Фильтры по дате/rate/code и маппинг в NdsDto")
  void filterAndMapping_shouldReturnOnlyMatchedAndMapped() {
    final var date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");

    // Датасет для проверки isBefore/isAfter + rate/code
    var pass1 = nds("1", "5", "A",
        date.minusDays(1), null);                 // проходит (start < date, end == null)
    var dropByEnd = nds("2", "10", "B",
        date.minusDays(10), date.minusDays(1));   // не проходит (end < date)
    var dropByStart = nds("3", "5", "C",
        date.plusDays(1), null);                  // не проходит (start > date)
    var pass2 = nds("4", "5", "B",
        date.minusDays(5), date.plusDays(5));     // проходит

    List<NdsFullDto> all = List.of(pass1, dropByEnd, dropByStart, pass2);

    // batch → всегда miss → loader отдаёт бакеты, собранные внутри сервиса
    doAnswer(inv -> {
      @SuppressWarnings("unchecked")
      Function<List<String>, List<Object>> loader = (Function<List<String>, List<Object>>) inv.getArgument(2);
      List<String> keys = inv.getArgument(1);
      return loader.apply(keys);
    }).when(batchLoad).fetchBatch(anyString(), anyList(), any(), any(), any());

    try (MockedStatic<BaseMasterDataRequestService> statics =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {

      when(baseService.requestData(any(ItemsSearchCriteriaRequest.class), eq(Context.BOOK)))
          .thenReturn(new GetItemsSearchResponse());
      statics.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(all);

      // 1) без фильтров rate/code → должны остаться 1 и 4
      var r1 = service.getBasicVatRate(date, null, null).getData();
      assertThat(r1).extracting(NdsDto::getId).containsExactlyInAnyOrder("1", "4");

      // 2) фильтр по rate=5 → остаются те же 1 и 4
      var r2 = service.getBasicVatRate(date, null, List.of("5")).getData();
      assertThat(r2).extracting(NdsDto::getId).containsExactlyInAnyOrder("1", "4");

      // 3) фильтр по code=B → только 4
      var r3 = service.getBasicVatRate(date, List.of("B"), null).getData();
      assertThat(r3).extracting(NdsDto::getId).containsExactly("4");

      // 4) проверим маппинг (toDto): поле rate/code/name/id переносятся как есть
      var any = r2.stream().filter(d -> d.getId().equals("4")).findFirst().orElseThrow();
      assertThat(any.getRate()).isEqualTo("5");
      assertThat(any.getCode()).isEqualTo("B");
      assertThat(any.getName()).isEqualTo("N4");
    }
  }

  // --- helpers ---

  private static NdsFullDto nds(String id, String rate, String code,
                                ZonedDateTime start, ZonedDateTime end) {
    // используйте ваш билдер/конструктор NdsFullDto
    return NdsFullDto.builder()
        .id(id)
        .name("N" + id)
        .rate(rate)
        .code(code)
        .rateDateStartZoned(start)
        .rateDateEndZoned(end)
        .build();
  }
}


=====================================
/package ru.sber.cs.supplier.portal.masterdata.services.impl.nds;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static ru.sber.cs.supplier.portal.masterdata.services.impl.nds.NdsService2.CACHE_NDS_BY_RATE;
import static ru.sber.cs.supplier.portal.masterdata.services.impl.nds.CacheAssertions.*;

import com.github.benmanes.caffeine.cache.Caffeine;
import java.time.Duration;
import java.time.ZonedDateTime;
import java.util.List;
import java.util.function.Function;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.mockito.MockedStatic;
import org.mockito.ArgumentCaptor;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cache.CacheManager;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.cache.annotation.EnableCaching;

import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;

import ru.sber.cs.supplier.portal.masterdata.models.adapter.dto.NdsFullDto;
import ru.sber.cs.supplier.portal.masterdata.services.BaseMasterDataRequestService;
import ru.sber.cs.supplier.portal.masterdata.services.BaseMasterDataRequestService.Context;
import ru.sber.cs.supplier.portal.masterdata.services.support.BatchCacheSupport;
import ru.sber.cs.supplier.portal.masterdata.models.adapter.request.ItemsSearchCriteriaRequest;
import ru.sber.cs.supplier.portal.masterdata.models.adapter.response.GetItemsSearchResponse;
import ru.sber.cs.supplier.portal.masterdata.mappers.NdsMapper;
import ru.sber.cs.supplier.portal.masterdata.properties.SearchRequestProperties;

/**
 * Интеграционный тест кеширования NdsService2 с Caffeine и BatchCacheSupport.
 */
@SpringBootTest(classes = {
    NdsService2CacheTest.TestCacheConfig.class,
    NdsService2.class
})
class NdsService2CacheTest {

  @TestConfiguration
  @EnableCaching
  static class TestCacheConfig {
    @Bean
    CacheManager cacheManager() {
      final var mgr = new CaffeineCacheManager(CACHE_NDS_BY_RATE);
      mgr.setCaffeine(Caffeine.newBuilder().maximumSize(100));
      return mgr;
    }
    @Bean
    BatchCacheSupport batchCacheSupport(final CacheManager cm) {
      return new BatchCacheSupport(cm);
    }
  }

  @Autowired private NdsService2 service;
  @Autowired private CacheManager cacheManager;

  @MockitoBean  private BaseMasterDataRequestService baseService;
  @MockitoBean  private NdsMapper ndsMapper;                  // нужен для ctor
  @MockitoBean  private SearchRequestProperties properties;

  // ВАЖНО: настоящий бин, но "под наблюдением"
  @MockitoSpyBean private BatchCacheSupport batchCacheSupport;

  private void stubProps() {
    when(properties.getSlugValueForVat()).thenReturn("vat");
    when(properties.getAttributeIdForTaxRateType()).thenReturn("type");
    when(properties.getAttributeIdForTaxRateActive()).thenReturn("active");
    when(properties.getAttributeIdForTaxRateEndDate()).thenReturn("end");
    when(properties.getAttributeIdForTaxRateStartDate()).thenReturn("start");
  }

  @Test
  @DisplayName("Warm-up → ключ появляется; повторный вызов — hit; cleanCache очищает")
  void warmup_hit_evict() {
    stubProps();
    final var date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");

    final var items = List.of(
        NdsFullDto.builder()
            .id("1").name("N1").rate("5").code("A")
            .rateDateStartZoned(date.minusDays(1))
            .rateDateEndZoned(null)
            .build()
    );

    // Захват ключей, с которыми вызвали батч
    final var keysCap = ArgumentCaptor.forClass(List.class);
    doCallRealMethod().when(batchCacheSupport)
        .fetchBatch(eq(CACHE_NDS_BY_RATE), keysCap.capture(), any(), any(), eq(RateBucket.class));

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {

      when(baseService.requestData(any(ItemsSearchCriteriaRequest.class), eq(Context.BOOK)))
          .thenReturn(new GetItemsSearchResponse());
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(items);

      // прогрев кеша
      service.getBasicVatRate(date, null, List.of("5"));

      // батч действительно вызывался и с ожидаемым ключом
      verify(batchCacheSupport, atLeastOnce()).fetchBatch(
          eq(CACHE_NDS_BY_RATE), anyList(), any(), any(), eq(RateBucket.class));
      assertThat(keysCap.getAllValues().get(0)).containsExactly("5");

      assertCacheHasKeys(cacheManager, CACHE_NDS_BY_RATE, "5");
      assertBucketSize(cacheManager, CACHE_NDS_BY_RATE, "5", 1);
      verify(baseService, times(1)).requestData(any(), eq(Context.BOOK));

      // повторный вызов по тому же rate — hit (без нового похода)
      service.getBasicVatRate(date, null, List.of("5"));
      verify(baseService, times(1)).requestData(any(), eq(Context.BOOK));

      // очистка кеша
      service.cleanCache();
      assertCacheEmpty(cacheManager, CACHE_NDS_BY_RATE);

      // снова прогрев → второй поход в МД
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(items);
      service.getBasicVatRate(date, null, List.of("5"));
      verify(baseService, times(2)).requestData(any(), eq(Context.BOOK));
    }
  }

  @Test
  @DisplayName("Без rate создаётся только бакет '__ALL__'")
  void no_rate_creates_all_bucket() {
    stubProps();
    final var date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");

    final var items = List.of(
        NdsFullDto.builder().id("1").name("N1").rate("5").code("A")
            .rateDateStartZoned(date.minusDays(5)).build(),
        NdsFullDto.builder().id("2").name("N2").rate("10").code("B")
            .rateDateStartZoned(date.minusDays(5)).build()
    );

    final var keysCap = ArgumentCaptor.forClass(List.class);
    doCallRealMethod().when(batchCacheSupport)
        .fetchBatch(eq(CACHE_NDS_BY_RATE), keysCap.capture(), any(), any(), eq(RateBucket.class));

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {

      when(baseService.requestData(any(ItemsSearchCriteriaRequest.class), eq(Context.BOOK)))
          .thenReturn(new GetItemsSearchResponse());
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(items);

      service.getBasicVatRate(date, null, null);

      verify(batchCacheSupport, atLeastOnce()).fetchBatch(
          eq(CACHE_NDS_BY_RATE), anyList(), any(), any(), eq(RateBucket.class));
      assertThat(keysCap.getAllValues().get(0)).containsExactly("__ALL__");

      assertCacheHasKeys(cacheManager, CACHE_NDS_BY_RATE, "__ALL__");
      assertBucketContainsIds(cacheManager, CACHE_NDS_BY_RATE, "__ALL__", "1", "2");
    }
  }

  @Test
  @DisplayName("Новый rate → добавляется новый бакет в кеше")
  void new_rate_adds_new_bucket() {
    stubProps();
    final var date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");

    final var items5 = List.of(
        NdsFullDto.builder().id("1").name("N1").rate("5").code("A")
            .rateDateStartZoned(date.minusDays(5)).build()
    );
    final var items10 = List.of(
        NdsFullDto.builder().id("2").name("N2").rate("10").code("B")
            .rateDateStartZoned(date.minusDays(5)).build()
    );

    final var keysCap = ArgumentCaptor.forClass(List.class);
    doCallRealMethod().when(batchCacheSupport)
        .fetchBatch(eq(CACHE_NDS_BY_RATE), keysCap.capture(), any(), any(), eq(RateBucket.class));

    try (MockedStatic<BaseMasterDataRequestService> st =
             Mockito.mockStatic(BaseMasterDataRequestService.class, Mockito.CALLS_REAL_METHODS)) {

      when(baseService.requestData(any(ItemsSearchCriteriaRequest.class), eq(Context.BOOK)))
          .thenReturn(new GetItemsSearchResponse());

      // rate=5
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(items5);
      service.getBasicVatRate(date, null, List.of("5"));
      assertCacheHasKeys(cacheManager, CACHE_NDS_BY_RATE, "5");

      // rate=10
      st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(any(), any()))
          .thenReturn(items10);
      service.getBasicVatRate(date, null, List.of("10"));

      // две фиксации keys: ["5"], ["10"]
      final var allKeysCaptured = keysCap.getAllValues();
      assertThat(allKeysCaptured).hasSizeGreaterThanOrEqualTo(2);
      assertThat(allKeysCaptured.get(0)).containsExactly("5");
      assertThat(allKeysCaptured.get(1)).containsExactly("10");

      assertCacheHasKeys(cacheManager, CACHE_NDS_BY_RATE, "5", "10");
      assertBucketContainsIds(cacheManager, CACHE_NDS_BY_RATE, "10", "2");
    }
  }
}

```
