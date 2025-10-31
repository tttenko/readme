```java
package your.package.nds;

import jakarta.annotation.Nonnull;
import jakarta.annotation.Nullable;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.tuple.Pair;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.stereotype.Service;
import org.springframework.util.CollectionUtils;

import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.concurrent.CopyOnWriteArraySet;
import java.util.stream.Stream;

/**
 * Сервис НДС: Spring Cache (Caffeine) + BatchCacheSupport.
 * - Данные грузим из МД одной батч-операцией без фильтра по rate.
 * - В кэше храним «корзины» по ключу (rate или "__ALL__").
 * - Фильтрация по дате/rate/code — как в старой реализации.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class NdsService {

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

    /** Элемент кэша: «корзина» по ключу (ставка или "__ALL__") */
    private record RateBucket(@Nonnull String key, @Nonnull Set<NdsFullDto> items) {}

    /**
     * Публичный API — поведение идентично старому:
     * 1) пытаемся прочитать из кэша; 2) промахи — батч-догрузка; 3) фильтр по дате/кодам/ставкам; 4) DTO.
     */
    @Nonnull
    public ResultObj<List<NdsDto>> getBasicVatRate(@Nullable final ZonedDateTime date,
                                                   @Nullable final List<String> code,
                                                   @Nullable final List<String> rate) {
        final ZonedDateTime targetDate = (date != null) ? date : ZonedDateTime.now();

        // какие ключи нужны из кэша?
        final List<String> keys = CollectionUtils.isEmpty(rate) ? List.of(ALL_KEY) : new ArrayList<>(rate);

        // батч-чтение/догрузка промахов (один вызов загрузчика на весь miss)
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
                        ))
                .toList();

        // используем готовый статический хелпер проекта
        return BaseMasterDataRequestService.getSuccessResponse(dto);
    }

    /* -----------------------------------------------------------
     *               Приватные шаги (stream/filters/map)
     * ----------------------------------------------------------- */

    /** 1) плоский поток доменных элементов из корзин */
    @Nonnull
    private Stream<NdsFullDto> asStream(@Nonnull final List<RateBucket> buckets) {
        return buckets.stream()
                .filter(Objects::nonNull)
                .flatMap(b -> b.items().stream());
    }

    /** 2) фильтрация по дате + (опц.) по rate и code */
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

    /** 3) маппинг в DTO */
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

    /* -----------------------------------------------------------
     *                   Загрузка и разбиение
     * ----------------------------------------------------------- */

    /**
     * Загружаем из МД один раз (без фильтра по rate), затем делим данные на корзины:
     * - если среди miss есть "__ALL__", создаём корзину "__ALL__" со всеми элементами;
     * - для остальных ключей создаём корзины по ставке.
     */
    @Nonnull
    private List<RateBucket> loadAllFromMdAndPartition(@Nonnull final ZonedDateTime date,
                                                       @Nonnull final List<String> missKeys) {
        final String dateString = date.format(DATE_TIME_FORMATTER);

        final var response = baseMasterDataRequestService.requestData(
                buildRequest(dateString),
                SearchRequestProperties.Context.BOOK
        );

        final List<NdsFullDto> all = baseMasterDataRequestService
                .createResultWithAttribute(response, ndsMapper);

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
                out.add(new RateBucket(miss,
                        new CopyOnWriteArraySet<>(byRate.getOrDefault(miss, Set.of()))));
            }
        }
        return out;
    }

    /* -----------------------------------------------------------
     *                 Построение запроса в МД
     * ----------------------------------------------------------- */

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

    /* -----------------------------------------------------------
     *                   Вспомогательные утилиты
     * ----------------------------------------------------------- */

    private static boolean isAfter(@Nonnull final ZonedDateTime date, @Nonnull final NdsFullDto e) {
        return e.getRateDateEndZoned() == null || e.getRateDateEndZoned().isAfter(date);
    }

    private static boolean isBefore(@Nonnull final ZonedDateTime date, @Nonnull final NdsFullDto e) {
        return e.getRateDateStartZoned() != null && e.getRateDateStartZoned().isBefore(date);
    }

    /** Очистка кеша (для оперподдержки/тестов). */
    @CacheEvict(cacheNames = CACHE_NDS_BY_RATE, allEntries = true)
    public void cleanCache() {
        log.info("NDS cache cleared");
    }
}



```
