```java

// com.example.supplier.SupplierService.java
package com.example.supplier;

import jakarta.annotation.Nullable;
import jakarta.annotation.Nonnull;
import jakarta.validation.constraints.NotNull;
import java.util.*;
import java.util.function.Function;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class SupplierService extends BaseMasterDataRequestService {

  // Имя полей фильтров в мастер-данных (как в старом коде)
  private static final String SUPPLIER_ID = "SupplierId";
  private static final String INN_KPP = "Inn+Kpp";

  // Имена кэшей (как в TerBankService2 — явные константы)
  public static final String SUPPLIER_REQ_BY_SUPPLIER_ID = "supplier_req_by_supplier_id";
  public static final String SUPPLIER_BY_ID = "supplier_by_id";
  public static final String SUPPLIER_BY_INN_KPP = "supplier_by_inn_kpp";

  private final SupplierRequisiteMapper supplierRequisiteMapper;
  private final SupplierMapper supplierMapper;
  private final BatchCacheSupport batchLoad;

  // --- API ---

  /** Поиск реквизитов поставщика по списку идентификаторов. */
  @Nonnull
  public ResultObj<List<BankDto>> searchSupplierRequisite(@Nonnull final List<String> ids) {
    final var list = batchLoad.fetchBatch(
        SUPPLIER_REQ_BY_SUPPLIER_ID,
        ids,
        this::loadRequisitesBySupplierIds,         // batch-loader из MD
        BankDto::getSupplierId                     // keyExtractor => SupplierId
    );
    return getSuccessResponse(list);
  }

  /** Поиск контрагентов по INN/KPP. Кэш — только если есть оба параметра (уникальный ключ). */
  @Nonnull
  public ResultObj<List<CounterpartyDto>> searchCounterpartiesByCriteria(@Nullable final String inn,
                                                                         @Nullable final String kpp) {
    final var criteria = buildCriteriaMap(inn, kpp);

    // если неуникально (только INN или только KPP) — без кэша, оставляем прежнюю семантику
    if (criteria.size() != 2) {
      return getSupplierByCriteria(criteria);
    }

    final String key = buildInnKppKey(inn, kpp);

    // батч-кэш на ключ inn:kpp, загрузчик игнорирует список ключей и дергает MD по criteria
    final List<CounterpartyDto> list = batchLoad.fetchBatch(
        SUPPLIER_BY_INN_KPP,
        List.of(key),
        missed -> loadSuppliersByCriteria(criteria), // вернёт 0..1 элемента
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

  // --- Вспомогательные методы загрузки из мастер-данных (оставлена прежняя логика маппинга) ---

  @Nonnull
  private List<BankDto> loadRequisitesBySupplierIds(@Nonnull final List<String> ids) {
    final GetItemsSearchResponse response = requestDataWithRefItemSlug2(
        properties.getSlugValueForSupplierRequisite(),               // словарь реквизитов
        properties.getAttributeIdForSupplierRequisite(),             // refItemSlug
        ids
    );
    return createResultWithAttribute(response, supplierRequisiteMapper);
  }

  @Nonnull
  private List<CounterpartyDto> loadSuppliersByIds(@Nonnull final List<String> ids) {
    final GetItemsSearchResponse response = requestDataWithAttribute(
        properties.getSlugValueForCounterparty(),
        ids,
        SearchRequestProperties.Context.BOOK
    );
    return createResultWithAttribute(response, supplierMapper);
  }

  @Nonnull
  private List<CounterpartyDto> loadSuppliersByCriteria(@Nonnull final Map<String, List<String>> criteria) {
    final GetItemsSearchResponse response = requestDataWithAttribute(
        properties.getSlugValueForCounterparty(),
        criteria
    );
    return createResultWithAttribute(response, supplierMapper);
  }

  @Nonnull
  private ResultObj<List<CounterpartyDto>> getSupplierByCriteria(@Nonnull final Map<String, List<String>> criteria) {
    final var list = loadSuppliersByCriteria(criteria);
    return getSuccessResponse(list);
  }

  // построение карты критериев (как раньше)
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

  // единый формат ключа кэша
  @Nonnull
  private static String buildInnKppKey(@Nullable final String inn, @Nullable final String kpp) {
    return String.format("inn:%s:kpp:%s", String.valueOf(inn), String.valueOf(kpp));
  }
}


```
