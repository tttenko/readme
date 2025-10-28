```java

package com.example.supplier;

import jakarta.annotation.Nonnull;
import jakarta.annotation.Nullable;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.function.Function;

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

import static com.example.masterdata.BaseMasterDataRequestService.createResultWithAttribute;
import static com.example.masterdata.BaseMasterDataRequestService.getSuccessResponse;

@Slf4j
@Service
@RequiredArgsConstructor
public class SupplierService {

  // Имена кэшей
  public static final String SUPPLIER_REQ_BY_SUPPLIER_ID = "supplier_req_by_supplier_id";
  public static final String SUPPLIER_BY_ID = "supplier_by_id";
  public static final String SUPPLIER_BY_INN_KPP = "supplier_by_inn_kpp";

  private final BatchCacheSupport batchLoad;
  private final SupplierRequisiteMapper supplierRequisiteMapper;
  private final SupplierMapper supplierMapper;

  // Внедряем вместо наследования
  private final BaseMasterDataRequestService base;
  private final SearchRequestProperties properties;

  /** Поиск реквизитов поставщика по списку идентификаторов. */
  @Nonnull
  public ResultObj<List<BankDto>> searchSupplierRequisite(@Nonnull final List<String> ids) {
    final var list = batchLoad.fetchBatch(
        SUPPLIER_REQ_BY_SUPPLIER_ID,
        ids,
        this::loadRequisitesBySupplierIds,
        BankDto::getSupplierId
    );
    return getSuccessResponse(list);
  }

  /** Поиск контрагентов по INN/KPP. Кэш включаем только при наличии обоих параметров. */
  @Nonnull
  public ResultObj<List<CounterpartyDto>> searchCounterpartiesByCriteria(@Nullable final String inn,
                                                                         @Nullable final String kpp) {
    final var criteria = buildCriteriaMap(inn, kpp);

    // Неуникальные запросы (только INN или только KPP) — без кэша, как и раньше
    if (criteria.size() != 2) {
      return getSupplierByCriteria(criteria);
    }

    final String key = buildInnKppKey(inn, kpp);
    final var list = batchLoad.fetchBatch(
        SUPPLIER_BY_INN_KPP,
        List.of(key),
        missed -> loadSuppliersByCriteria(criteria),
        dto -> buildInnKppKey(dto.getInn(), dto.getKpp())
    );
    return getSuccessResponse(list);
  }

  /** Получение контрагентов по списку идентификаторов. */
  @Nonnull
  public ResultObj<List<CounterpartyDto>> getCounterpartiesById(@Nonnull final List<String> ids) {
    final var list = batchLoad.fetchBatch(
        SUPPLIER_BY_ID,
        ids,
        this::loadSuppliersByIds,
        CounterpartyDto::getId
    );
    return getSuccessResponse(list);
  }

  // ----------------------- Загрузчики из мастер-данных -----------------------

  @Nonnull
  private List<BankDto> loadRequisitesBySupplierIds(@Nonnull final List<String> ids) {
    final GetItemsSearchResponse resp = base.requestDataWithRefItemSlug2(
        properties.getSlugValueForSupplierRequisite(),
        properties.getAttributeIdForSupplierRequisite(),
        ids
    );
    return BaseMasterDataRequestService.createWithAttribute(resp, supplierRequisiteMapper);
  }

  @Nonnull
  private List<CounterpartyDto> loadSuppliersByIds(@Nonnull final List<String> ids) {
    final GetItemsSearchResponse resp = base.requestDataWithAttribute(
        properties.getSlugValueForCounterparty(),
        ids,
        SearchRequestProperties.Context.BOOK
    );
    return BaseMasterDataRequestService.createWithAttribute(resp, supplierMapper);
  }

  @Nonnull
  private List<CounterpartyDto> loadSuppliersByCriteria(@Nonnull final Map<String, List<String>> criteria) {
    final GetItemsSearchResponse resp = base.requestDataWithAttribute(
        properties.getSlugValueForCounterparty(),
        criteria
    );
    return BaseMasterDataRequestService.createWithAttribute(resp, supplierMapper);
  }

  @Nonnull
  private ResultObj<List<CounterpartyDto>> getSupplierByCriteria(@Nonnull final Map<String, List<String>> criteria) {
    return BaseMasterDataRequestService.createResultObjWithAttribute(
        base.requestDataWithAttribute(properties.getSlugValueForCounterparty(), criteria),
        supplierMapper
    );
  }

  // ----------------------- Утилиты -----------------------

  @Nonnull
  private Map<String, List<String>> buildCriteriaMap(@Nullable final String inn, @Nullable final String kpp) {
    final Map<String, List<String>> criteria = new HashMap<>(2);
    if (inn != null && !inn.isBlank()) {
      criteria.put(properties.getAttributeForInn(), List.of(inn));
    }
    if (kpp != null && !kpp.isBlank()) {
      criteria.put(properties.



```
