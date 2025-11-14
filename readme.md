```java

@Component
@RequiredArgsConstructor
public class SupplierCriteriaBuilder {

    private final SearchRequestProperties properties;

    /**
     * Строит карту критериев для поиска контрагентов по ИНН/КПП.
     */
    @NonNull
    public Map<String, List<String>> buildCriteria(
            @Nullable String inn,
            @Nullable String kpp
    ) {
        Map<String, List<String>> criteria = new HashMap<>(2);

        if (inn != null && !inn.isBlank()) {
            criteria.put(properties.getAttributeIdForInn(), List.of(inn));
        }
        if (kpp != null && !kpp.isBlank()) {
            criteria.put(properties.getAttributeIdForKpp(), List.of(kpp));
        }

        return criteria;
    }

    /**
     * Формирует детерминированный ключ кеша по паре ИНН/КПП.
     */
    @NonNull
    public String buildInnKppKey(@Nullable String inn, @Nullable String kpp) {
        return String.format("inn:%s:kpp:%s", inn, kpp);
    }

    /**
     * Восстанавливает criteria из строкового ключа кеша.
     * Ожидаемый формат ключа: "inn:<ИНН>:kpp:<КПП>".
     */
    @NonNull
    public Map<String, List<String>> buildCriteriaFromKey(@NonNull String key) {
        String inn = null;
        String kpp = null;

        if (!key.isBlank()) {
            String[] parts = key.split(":", 4);
            if (parts.length == 4) {
                inn = parts[1];
                kpp = parts[3];
            }
        }

        return buildCriteria(inn, kpp);
    }
}


@Slf4j
@Service
@RequiredArgsConstructor
public class SupplierService2 {

    public static final String SUPPLIER_REQ_BY_SUPPLIER_ID = "supplier_req_by_supplier_id";
    public static final String SUPPLIER_BY_ID              = "supplier_by_id";
    public static final String SUPPLIER_BY_INN_KPP         = "supplier_by_inn_kpp";

    private final SupplierCacheOps supplierCache;
    private final CacheGetOrLoadService cacheGetOrLoadService;

    private final SupplierMapper supplierMapper;
    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SearchRequestProperties properties;
    private final SupplierCriteriaBuilder criteriaBuilder;

    /**
     * Возвращает реквизиты поставщика по списку идентификаторов.
     */
    @NonNull
    public ResultObj<List<BankDto>> searchSupplierRequisite(
            @NonNull List<String> ids
    ) {
        List<BankDto> response = ids.stream()
                .filter(id -> id != null && !id.isBlank())
                .map(supplierCache::loadBySupplierId)
                .flatMap(List::stream)
                .toList();

        return getSuccessResponse(response);
    }

    /**
     * Возвращает список контрагентов по критериям (ИНН и/или КПП).
     * Если заданы не оба параметра — идём напрямую в мастер-данные без кеша.
     * Если заданы оба — используем кеш SUPPLIER_BY_INN_KPP через CacheGetOrLoadService.
     */
    @NonNull
    public ResultObj<List<CounterpartyDto>> searchCounterpartiesByCriteria(
            @Nullable String inn,
            @Nullable String kpp
    ) {
        Map<String, List<String>> criteria = criteriaBuilder.buildCriteria(inn, kpp);

        if (criteria.size() != 2) {
            // поведение как раньше: прямой поиск без кеша
            return getSupplierByCriteria(criteria);
        }

        String key = criteriaBuilder.buildInnKppKey(inn, kpp);

        List<CounterpartyDto> list =
                cacheGetOrLoadService.<CounterpartyDto>fetchData(SUPPLIER_BY_INN_KPP, List.of(key));

        return getSuccessResponse(list);
    }

    /**
     * Возвращает контрагентов по списку идентификаторов, с кешированием по SUPPLIER_BY_ID.
     */
    @NonNull
    public ResultObj<List<CounterpartyDto>> getCounterpartiesById(
            @NonNull List<String> ids
    ) {
        List<CounterpartyDto> list =
                cacheGetOrLoadService.<CounterpartyDto>fetchData(SUPPLIER_BY_ID, ids);

        return getSuccessResponse(list);
    }

    /**
     * Прямой поиск контрагентов по критериям без кеша.
     */
    @NonNull
    private ResultObj<List<CounterpartyDto>> getSupplierByCriteria(
            @NonNull Map<String, List<String>> criteria
    ) {
        return createResultObjWithAttribute(
                baseMasterDataRequestService.requestDataWithAttribute(
                        properties.getSlugValueForCounterparty(),
                        criteria
                ),
                supplierMapper
        );
    }
}


@Component
@RequiredArgsConstructor
public class LoaderSupplierById implements BatchLoader<CounterpartyDto> {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SupplierMapper supplierMapper;
    private final SearchRequestProperties properties;

    @Override
    public String cacheName() {
        return SupplierService2.SUPPLIER_BY_ID;
    }

    @Override
    public Class<CounterpartyDto> elementType() {
        return CounterpartyDto.class;
    }

    @Override
    public String extractKey(CounterpartyDto dto) {
        return dto.getId();
    }

    @Override
    @NonNull
    public List<CounterpartyDto> fetchByKeys(@NonNull List<String> ids) {
        if (ids.isEmpty()) {
            return List.of();
        }

        GetItemsSearchResponse resp =
                baseMasterDataRequestService.requestDataWithAttribute(
                        properties.getSlugValueForCounterparty(),
                        ids,
                        SearchRequestProperties.Context.BOOK
                );

        return BaseMasterDataRequestService.createResultWithAttribute(resp, supplierMapper);
    }
}


@Component
@RequiredArgsConstructor
public class LoaderSupplierByInnKpp implements BatchLoader<CounterpartyDto> {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SupplierMapper supplierMapper;
    private final SearchRequestProperties properties;
    private final SupplierCriteriaBuilder criteriaBuilder;

    @Override
    public String cacheName() {
        return SupplierService2.SUPPLIER_BY_INN_KPP;
    }

    @Override
    public Class<CounterpartyDto> elementType() {
        return CounterpartyDto.class;
    }

    @Override
    public String extractKey(CounterpartyDto dto) {
        // формат ключа строго совпадает с тем, что используется в SupplierService2
        return criteriaBuilder.buildInnKppKey(dto.getInn(), dto.getKpp());
    }

    @Override
    @NonNull
    public List<CounterpartyDto> fetchByKeys(@NonNull List<String> keys) {
        if (keys.isEmpty()) {
            return List.of();
        }

        // В текущей логике SupplierService2 всегда передаёт один ключ,
        // но даже если их будет несколько — можно расширить реализацию позже.
        String key = keys.get(0);

        Map<String, List<String>> criteria = criteriaBuilder.buildCriteriaFromKey(key);
        if (criteria.isEmpty()) {
            return List.of();
        }

        GetItemsSearchResponse resp =
                baseMasterDataRequestService.requestDataWithAttribute(
                        properties.getSlugValueForCounterparty(),
                        criteria
                );

        return BaseMasterDataRequestService.createResultWithAttribute(resp, supplierMapper);
    }
}


@Component
@RequiredArgsConstructor
public class LoaderSupplierByCriteria {

    private final BaseMasterDataRequestService baseMasterDataRequestService;
    private final SupplierMapper supplierMapper;
    private final SearchRequestProperties properties;

    /**
     * Прямой поиск контрагентов в мастер-данных по критериям (ИНН/КПП) без кеша.
     */
    @NonNull
    public ResultObj<List<CounterpartyDto>> loadByCriteria(
            @NonNull Map<String, List<String>> criteria
    ) {
        return createResultObjWithAttribute(
                baseMasterDataRequestService.requestDataWithAttribute(
                        properties.getSlugValueForCounterparty(),
                        criteria
                ),
                supplierMapper
        );
    }
}


```
