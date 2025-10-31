```java
package com.example.nds;

import com.example.common.BaseMasterDataRequestService;
import com.example.common.SearchRequestProperties;
import com.example.common.requests.ItemsSearchCriteriaRequest;
import com.example.common.requests.RequestFactory;
import com.example.common.responses.GetItemsSearchResponse;
import com.example.nds.dto.NdsDto;
import com.example.nds.dto.NdsFullDto;
import com.example.nds.mapper.NdsMapper;
import com.example.web.ResultObj;
import jakarta.annotation.Nonnull;
import jakarta.annotation.Nullable;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.stereotype.Service;
import org.springframework.util.CollectionUtils;

import java.io.Serializable;
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.concurrent.CopyOnWriteArraySet;

import static java.util.Objects.nonNull;

/**
 * NdsService — использует Spring Cache (Caffeine) и BatchCacheSupport для батч-загрузки
 * и кэширования значений НДС по ключам rate (и специальный ключ "__ALL__" при пустом запросе rate).
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class NdsService {

  public static final String CACHE_NDS_BY_RATE = "nds_by_rate";
  private static final String ALL_KEY = "__ALL__";
  private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mmXXX");

  private static final List<String> BASIC_RATE_TYPE = List.of("1");
  private static final List<String> ACTIVE_RATE = List.of("true");

  private final NdsMapper ndsMapper;
  private final BatchCacheSupport batchLoad;
  private final BaseMasterDataRequestService baseMasterDataRequestService;
  private final SearchRequestProperties properties;

  // record для кешируемых «корзин»: ключ (rate | "__ALL__") + набор элементов
  private record RateBucket(@Nonnull String key, @Nonnull Set<NdsFullDto> items) {}

  /** Публичный API — аналог старого getBasicVatRate, поведение сохранено. */
  @Nonnull
  public ResultObj<List<NdsDto>> getBasicVatRate(@Nullable final ZonedDateTime date,
                                                 @Nullable final List<String> code,
                                                 @Nullable final List<String> rate) {
    final ZonedDateTime targetDate = date != null ? date : ZonedDateTime.now();

    // 1) какие ключи нужны из кеша?
    final List<String> keys = (CollectionUtils.isEmpty(rate) ? List.of(ALL_KEY) : new ArrayList<>(rate));

    // 2) батч-чтение/догрузка промахов (одна поездка в МД для всех miss)
    final List<RateBucket> buckets = batchLoad.fetchBatch(
        CACHE_NDS_BY_RATE,
        keys,
        miss -> loadAllFromMdAndPartition(targetDate, miss),
        RateBucket::key,
        RateBucket.class
    );

    // 3) собираем результирующий список и фильтруем (даты/коды — идентично прежнему)
    final List<NdsFullDto> flat = buckets.stream()
        .filter(Objects::nonNull)
        .flatMap(b -> b.items().stream())
        .toList();

    final List<NdsFullDto> filtered = flat.stream()
        .filter(e -> isBefore(targetDate, e) && isAfter(targetDate, e))
        .filter(e -> CollectionUtils.isEmpty(rate) || rate.contains(e.getRate()))
        .filter(e -> CollectionUtils.isEmpty(code) || code.contains(e.getCode()))
        .toList();

    final var dto = filtered.stream()
        .map(e -> NdsDto.builder()
            .name(e.getName())
            .rate(e.getRate())
            .id(e.getId())
            .code(e.getCode())
            .name(e.getName())
            .build())
        .toList();

    return getSuccessResponse(dto);
  }

  /**
   * Загрузка ИЗ МД ОДИН РАЗ для всего miss-набора, без фильтра по rate
   * → распил по корзинам (miss содержит либо список rate, либо "__ALL__").
   */
  @Nonnull
  private List<RateBucket> loadAllFromMdAndPartition(@Nonnull final ZonedDateTime date,
                                                     @Nonnull final List<String> missKeys) {
    final String dateString = date.format(DATE_TIME_FORMATTER);

    final GetItemsSearchResponse response = baseMasterDataRequestService.requestData(
        buildRequest(dateString),
        SearchRequestProperties.Context.BOOK);

    final List<NdsFullDto> all = baseMasterDataRequestService.createResultWithAttribute(response, ndsMapper);

    // группируем по rate
    final Map<String, Set<NdsFullDto>> byRate = new HashMap<>();
    for (NdsFullDto e : all) {
      byRate.computeIfAbsent(e.getRate(), r -> new CopyOnWriteArraySet<>()).add(e);
    }

    final List<RateBucket> out = new ArrayList<>(missKeys.size());
    for (String miss : missKeys) {
      if (ALL_KEY.equals(miss)) {
        // полная корзина
        out.add(new RateBucket(ALL_KEY, new CopyOnWriteArraySet<>(all)));
      } else {
        final Set<NdsFullDto> set = byRate.getOrDefault(miss, Set.of());
        out.add(new RateBucket(miss, new CopyOnWriteArraySet<>(set)));
      }
    }
    return out;
  }

  // --- Очистка кеша (нужна тестам/операциям поддержки) ---
  @CacheEvict(cacheNames = CACHE_NDS_BY_RATE, allEntries = true)
  public void cleanCache() {
    log.info("NDS cache cleared");
  }

  // --- Опционально: статус кеша для отладки/админки ---
  @Nonnull
  public Map<String, Serializable> getCacheStatus() {
    // количество ключей (rate + "__ALL__") — оценка «внешнего размера»
    // (через CacheManager напрямую доставать map значений Caffeine нельзя — даём безопасный ответ)
    return Map.of("Кол-во записей", -1); // если нужна фактическая статистика — используйте метрики Caffeine/Actuator
  }

  // --- фильтры (перенос из старого кода) ---
  private static boolean isAfter(@Nonnull final ZonedDateTime date, @Nonnull final NdsFullDto e) {
    return !nonNull(e.getRateDateEndZoned()) || e.getRateDateEndZoned().isAfter(date);
  }
  private static boolean isBefore(@Nonnull final ZonedDateTime date, @Nonnull final NdsFullDto e) {
    return nonNull(e.getRateDateStartZoned()) && e.getRateDateStartZoned().isBefore(date);
  }

  // --- построение запроса в МД (без фильтра по rate — как раньше) ---
  @Nonnull
  private ItemsSearchCriteriaRequest buildRequest(@Nonnull final String dateString) {
    return RequestFactory.getByAttrValuesBuilder()
        .dictionaryName(properties.getSlugValueForVat())
        .addAttributesAndRefItemSlug(properties.getAttributeIdForTaxRateType(), BASIC_RATE_TYPE)
        .addAttributesAndValue(properties.getAttributeIdForTaxRateActive(), ACTIVE_RATE)
        .addAttributesAndValue(properties.getAttributeIdForTaxRateEndDate(), List.of(dateString), RequestFactory.MORE_OPERATION)
        .addAttributesAndValue(properties.getAttributeIdForTaxRateStartDate(), List.of(dateString), RequestFactory.LESS_OPERATION)
        .build();
  }

  // обёртка, чтобы вернуть ResultObj как в старом сервисе
  @Nonnull
  private <T> ResultObj<List<T>> getSuccessResponse(@Nonnull final List<T> data) {
    return ResultObj.<List<T>>builder().data(data).success(true).build();
  }
}



```
