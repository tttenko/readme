```java

package com.example.supplier;

import jakarta.annotation.Nonnull;
import jakarta.annotation.Nullable;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import com.example.cache.BatchCacheSupport;
import com.example.masterdata.BaseMasterDataRequestService;
import com.example.masterdata.GetItemsSearchResponse;
import com.example.masterdata.SearchRequestProperties;
import com.example.masterdata.dto.BankDto;
import com.example.masterdata.dto.CounterpartyDto;
import com.example.masterdata.dto.ResultObj;
import com.example.masterdata.mapper.SupplierMapper;
import com.example.masterdata.mapper.SupplierRequisiteMapper;

import static com.example.masterdata.BaseMasterDataRequestService.createResultObjWithAttribute;
import static com.example.masterdata.BaseMasterDataRequestService.createWithAttribute;
import static com.example.masterdata.BaseMasterDataRequestService.getSuccessResponse;

@Slf4j
@Service
@RequiredArgsConstructor
public class SupplierService {

  public static final String SUPPLIER_REQ_BY_SUPPLIER_ID = "supplier_req_by_supplier_id";
  public static final String SUPPLIER_BY_ID = "supplier_by_id";
  public static final String SUPPLIER_BY_INN_KPP = "supplier_by_inn_kpp";

  private final BatchCacheSupport batchLoad;
  private final SupplierRequisiteMapper supplierRequisiteMapper;
  private final SupplierMapper supplierMapper;

  private final BaseMasterDataRequestService base;
  private final SearchRequestProperties properties;

  @Nonnull
  public ResultObj<List<BankDto>> searchSupplierRequisite(@Nonnull final List<String> ids) {
    final var list = batchLoad.fetchBatch(
        SUPPLIER_REQ_BY_SUPPLIER_ID,
        ids,
        this::loadRequisitesBySupplierIds,
        BankDto::getSupplierId,
        BankDto.class
    );
    return getSuccessResponse(list);
  }

  @Nonnull
  public ResultObj<List<CounterpartyDto>> searchCounterpartiesByCriteria(@Nullable final String inn,
                                                                         @Nullable final String kpp) {
    final var criteria = buildCriteriaMap(inn, kpp);
    if (criteria.size() != 2) {
      return getSupplierByCriteria(criteria);
    }
    final String key = buildInnKppKey(inn, kpp);
    final var list = batchLoad.fetchBatch(
        SUPPLIER_BY_INN_KPP,
        List.of(key),
        missed -> loadSuppliersByCriteria(criteria),
        dto -> buildInnKppKey(dto.getInn(), dto.getKpp()),
        CounterpartyDto.class
    );
    return getSuccessResponse(list);
  }

  @Nonnull
  public ResultObj<List<CounterpartyDto>> getCounterpartiesById(@Nonnull final List<String> ids) {
    final var list = batchLoad.fetchBatch(
        SUPPLIER_BY_ID,
        ids,
        this::loadSuppliersByIds,
        CounterpartyDto::getId,
        CounterpartyDto.class
    );
    return getSuccessResponse(list);
  }

  // -------- Загрузчики из мастер-данных --------

  @Nonnull
  private List<BankDto> loadRequisitesBySupplierIds(@Nonnull final List<String> ids) {
    final GetItemsSearchResponse resp = base.requestDataWithRefItemSlug2(
        properties.getSlugValueForSupplierRequisite(),
        properties.getAttributeIdForSupplierRequisite(),
        ids
    );
    return createWithAttribute(resp, supplierRequisiteMapper);
  }

  @Nonnull
  private List<CounterpartyDto> loadSuppliersByIds(@Nonnull final List<String> ids) {
    final GetItemsSearchResponse resp = base.requestDataWithAttribute(
        properties.getSlugValueForCounterparty(),
        ids,
        SearchRequestProperties.Context.BOOK
    );
    return createWithAttribute(resp, supplierMapper);
  }

  @Nonnull
  private List<CounterpartyDto> loadSuppliersByCriteria(@Nonnull final Map<String, List<String>> criteria) {
    final GetItemsSearchResponse resp = base.requestDataWithAttribute(
        properties.getSlugValueForCounterparty(),
        criteria
    );
    return createWithAttribute(resp, supplierMapper);
  }

  @Nonnull
  private ResultObj<List<CounterpartyDto>> getSupplierByCriteria(@Nonnull final Map<String, List<String>> criteria) {
    return createResultObjWithAttribute(
        base.requestDataWithAttribute(properties.getSlugValueForCounterparty(), criteria),
        supplierMapper
    );
  }

  // -------- Утилиты --------

  @Nonnull
  private Map<String, List<String>> buildCriteriaMap(@Nullable final String inn, @Nullable final String kpp) {
    final Map<String, List<String>> criteria = new HashMap<>(2);
    if (inn != null && !inn.isBlank()) {
      criteria.put(properties.getAttributeForInn(), List.of(inn));
    }
    if (kpp != null && !kpp.isBlank()) {
      criteria.put(properties.getAttributeForKpp(), List.of(kpp));
    }
    return criteria;
  }

  @Nonnull
  private static String buildInnKppKey(@Nullable final String inn, @Nullable final String kpp) {
    return String.format("inn:%s:kpp:%s", String.valueOf(inn), String.valueOf(kpp));
  }
}







@Slf4j
@Component
@RequiredArgsConstructor
public class BatchCacheSupport {

    private final CacheManager cacheManager;

    @NonNull
    public <T> List<T> fetchBatch(
            @NonNull final String cacheName,
            @NonNull final List<String> keys,
            @NonNull final Function<List<String>, List<T>> loader,
            @NonNull final Function<T, String> keyExtractor,
            @NonNull final Class<T> type) {

        if (keys.isEmpty()) return List.of();

        final Cache cache = cacheManager.getCache(cacheName);

        final Map<String, T> hits = new LinkedHashMap<>();
        final List<String> miss = new ArrayList<>();

        collectHitsAndMisses(cache, keys, type, hits, miss);

        final List<T> loaded = miss.isEmpty() ? List.of() : safeLoadBatch(loader, miss);

        putLoadedToCache(cache, loaded, keyExtractor);

        return orderByOriginal(keys, hits, loaded, keyExtractor);
    }

    private static <T> List<T> safeLoadBatch(
            @NonNull final Function<List<String>, List<T>> loader,
            @NonNull final List<String> miss) {
        try {
            final List<T> res = loader.apply(miss);
            return (res == null) ? List.of() : res;
        } catch (RuntimeException ex) {
            return List.of();
        }
    }

    private <T> void collectHitsAndMisses(
            @Nullable final Cache cache,
            @NonNull final List<String> keys,
            @NonNull final Class<T> type,
            @NonNull final Map<String, T> hitsOut,
            @NonNull final List<String> missOut) {
        for (String key : keys) {
            if (key == null || key.isBlank()) {
                log.debug("Skip invalid cache key: '{}'", key);
                continue;
            }

            T cached = null;
            if (cache != null) {
                try {
                    cached = cache.get(key, type);
            }

            if (cached != null) hitsOut.put(key, cached);
            else missOut.add(key);
        }
    }

    private <T> void putLoadedToCache(
            @Nullable final Cache cache,
            @NonNull final List<T> loaded,
            @NonNull final Function<T, String> keyExtractor) {
        if (cache == null || loaded.isEmpty()) return;

        for (T item : loaded) {
            if (item == null) continue;

            final String key;
            try {
                key = keyExtractor.apply(item);
            } catch (RuntimeException ex) {
                log.warn("Failed to extract cache key for item {}: {}", item, ex.toString());
                continue;
            }

            if (key == null || key.isBlank()) {
                log.debug("Skip caching item with blank key: {}", item);
                continue;
            }
            cache.put(key, item);
        }
    }

    @NonNull
    private <T> List<T> orderByOriginal(
            @NonNull final List<String> originalKeys,
            @NonNull final Map<String, T> hits,
            @NonNull final List<T> loaded,
            @NonNull final Function<T, String> keyExtractor) {

        final Map<String, T> byKey = new HashMap<>(hits);
        for (T item : loaded) {
            if (item == null) continue;
            final String key;
            try {
                key = keyExtractor.apply(item);
            } catch (RuntimeException ex) {
                log.warn("Failed to extract key for ordering, item {}: {}", item, ex.toString());
                continue;
            }
            if (key == null || key.isBlank()) continue;
            byKey.put(key, item);
        }

        return originalKeys.stream()
                .map(byKey::get)
                .filter(Objects::nonNull)
                .toList();
    }
}


```
