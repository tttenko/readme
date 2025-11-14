```java

@ExtendWith(MockitoExtension.class)
class SupplierService2Test {

    @Mock
    private SupplierCacheOps supplierCache;

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    @Mock
    private HelperSupplierCriteriaBuilder criteriaBuilder;

    @Mock
    private LoaderSupplierByCriteria loaderSupplierByCriteria;

    @InjectMocks
    private SupplierService2 supplierService;

    // --------------------------------------------------------------
    // searchSupplierRequisite
    // --------------------------------------------------------------

    @Test
    void givenIdsWithBlanks_whenSearchSupplierRequisite_thenUseSupplierCacheOnlyForValidIds() {
        // given
        BankDto b1 = mock(BankDto.class);
        BankDto b2 = mock(BankDto.class);
        List<String> ids = Arrays.asList(" ", null, "S1", "");

        when(supplierCache.loadBySupplierId("S1"))
                .thenReturn(List.of(b1, b2));

        // when
        ResultObj<List<BankDto>> result = supplierService.searchSupplierRequisite(ids);

        // then
        verify(supplierCache, times(1)).loadBySupplierId("S1");
        verifyNoMoreInteractions(supplierCache);

        assertEquals(List.of(b1, b2), result.getData());
    }

    // --------------------------------------------------------------
    // searchCounterpartiesByCriteria: inn + kpp -> кеш
    // --------------------------------------------------------------

    @Test
    void givenInnAndKpp_whenSearchCounterpartiesByCriteria_thenUseCacheGetOrLoadService() {
        // given
        String inn = "7700000000";
        String kpp = "770001001";
        String key = "inn:7700000000:kpp:770001001";

        Map<String, List<String>> criteria = Map.of(
                "innAttr", List.of(inn),
                "kppAttr", List.of(kpp)
        );
        when(criteriaBuilder.buildCriteria(inn, kpp)).thenReturn(criteria);
        when(criteriaBuilder.buildInnKppKey(inn, kpp)).thenReturn(key);

        List<CounterpartyDto> cached = List.of(mock(CounterpartyDto.class));
        when(cacheGetOrLoadService.<CounterpartyDto>fetchData(
                eq(SupplierService2.SUPPLIER_BY_INN_KPP),
                eq(List.of(key))
        )).thenReturn(cached);

        // when
        ResultObj<List<CounterpartyDto>> result =
                supplierService.searchCounterpartiesByCriteria(inn, kpp);

        // then
        assertEquals(cached, result.getData());

        verify(criteriaBuilder).buildCriteria(inn, kpp);
        verify(criteriaBuilder).buildInnKppKey(inn, kpp);
        verify(cacheGetOrLoadService).fetchData(
                SupplierService2.SUPPLIER_BY_INN_KPP,
                List.of(key)
        );

        verifyNoInteractions(loaderSupplierByCriteria);
    }

    // --------------------------------------------------------------
    // searchCounterpartiesByCriteria: только inn/kpp -> прямой лоадер
    // --------------------------------------------------------------

    @Test
    void givenOnlyInn_whenSearchCounterpartiesByCriteria_thenBypassCacheAndCallLoaderByCriteria() {
        // given
        String inn = "7700000000";
        String kpp = null;

        Map<String, List<String>> criteria = Map.of(
                "innAttr", List.of(inn)
        );
        when(criteriaBuilder.buildCriteria(inn, kpp)).thenReturn(criteria);

        List<CounterpartyDto> direct = List.of(mock(CounterpartyDto.class));
        when(loaderSupplierByCriteria.loadByCriteria(criteria)).thenReturn(direct);

        // when
        ResultObj<List<CounterpartyDto>> result =
                supplierService.searchCounterpartiesByCriteria(inn, kpp);

        // then
        assertEquals(direct, result.getData());

        verify(criteriaBuilder).buildCriteria(inn, kpp);
        verify(loaderSupplierByCriteria).loadByCriteria(criteria);

        verifyNoInteractions(cacheGetOrLoadService);
        verify(criteriaBuilder, never()).buildInnKppKey(any(), any());
    }

    // --------------------------------------------------------------
    // getCounterpartiesById
    // --------------------------------------------------------------

    @Test
    void givenIds_whenGetCounterpartiesById_thenUseCacheGetOrLoadServiceWithSupplierByIdCache() {
        // given
        List<String> ids = List.of("A", "B");

        List<CounterpartyDto> list = List.of(
                mock(CounterpartyDto.class),
                mock(CounterpartyDto.class)
        );
        when(cacheGetOrLoadService.<CounterpartyDto>fetchData(
                eq(SupplierService2.SUPPLIER_BY_ID),
                eq(ids)
        )).thenReturn(list);

        // when
        ResultObj<List<CounterpartyDto>> result =
                supplierService.getCounterpartiesById(ids);

        // then
        assertEquals(list, result.getData());

        verify(cacheGetOrLoadService).fetchData(
                SupplierService2.SUPPLIER_BY_ID,
                ids
        );
        verifyNoInteractions(loaderSupplierByCriteria);
    }
}
```
