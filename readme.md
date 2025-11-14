```java




@ExtendWith(MockitoExtension.class)
class LoaderSupplierByInnKppTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SupplierMapper supplierMapper;

    @Mock
    private SearchRequestProperties properties;

    @Mock
    private HelperSupplierCriteriaBuilder criteriaBuilder;

    @InjectMocks
    private LoaderSupplierByInnKpp loader;

    // -------------------------------------------------------
    // Простые методы: cacheName / elementType / extractKey
    // -------------------------------------------------------

    @Test
    void givenLoader_whenCacheNameCalled_thenReturnSupplierByInnKppConstant() {
        // when
        String name = loader.cacheName();

        // then
        assertEquals(SupplierService2.SUPPLIER_BY_INN_KPP, name);
    }

    @Test
    void givenLoader_whenElementTypeCalled_thenReturnCounterpartyDtoClass() {
        // when
        Class<CounterpartyDto> type = loader.elementType();

        // then
        assertEquals(CounterpartyDto.class, type);
    }

    @Test
    void givenDto_whenExtractKeyCalled_thenUseCriteriaBuilderAndReturnKey() {
        // given
        CounterpartyDto dto = mock(CounterpartyDto.class);
        when(dto.getInn()).thenReturn("7700000000");
        when(dto.getKpp()).thenReturn("770001001");

        String expectedKey = "inn:7700000000:kpp:770001001";
        when(criteriaBuilder.buildInnKppKey("7700000000", "770001001"))
                .thenReturn(expectedKey);

        // when
        String key = loader.extractKey(dto);

        // then
        assertEquals(expectedKey, key);

        verify(dto).getInn();
        verify(dto).getKpp();
        verify(criteriaBuilder).buildInnKppKey("7700000000", "770001001");
    }

    // -------------------------------------------------------
    // fetchByKeys
    // -------------------------------------------------------

    @Test
    void givenEmptyKeys_whenFetchByKeys_thenReturnEmptyAndDoNotCallDependencies() {
        // when
        List<CounterpartyDto> result = loader.fetchByKeys(List.of());

        // then
        assertNotNull(result);
        assertTrue(result.isEmpty());

        verifyNoInteractions(baseMasterDataRequestService, properties, criteriaBuilder, supplierMapper);
    }

    @Test
    void givenKeys_whenCriteriaFromKeyIsEmpty_thenReturnEmptyAndDoNotCallBaseService() {
        // given
        String key = "inn:null:kpp:null";
        List<String> keys = List.of(key);

        when(criteriaBuilder.buildCriteriaFromKey(key))
                .thenReturn(Map.of()); // пустая map

        // when
        List<CounterpartyDto> result = loader.fetchByKeys(keys);

        // then
        assertNotNull(result);
        assertTrue(result.isEmpty());

        verify(criteriaBuilder).buildCriteriaFromKey(key);
        verifyNoInteractions(baseMasterDataRequestService, properties, supplierMapper);
    }

    @Test
    void givenKeysAndCriteria_whenFetchByKeys_thenCallBaseServiceAndMapResult() {
        // given
        String key = "inn:7700000000:kpp:770001001";
        List<String> keys = List.of(key);

        Map<String, List<String>> criteria = Map.of(
                "innAttr", List.of("7700000000"),
                "kppAttr", List.of("770001001")
        );
        when(criteriaBuilder.buildCriteriaFromKey(key)).thenReturn(criteria);

        String slug = "counterpartySlug";
        when(properties.getSlugValueForCounterparty()).thenReturn(slug);

        GetItemsSearchResponse resp = mock(GetItemsSearchResponse.class);
        when(baseMasterDataRequestService.requestDataWithAttribute(
                slug,
                criteria
        )).thenReturn(resp);

        List<CounterpartyDto> mapped = List.of(
                mock(CounterpartyDto.class),
                mock(CounterpartyDto.class)
        );

        // мок статического метода BaseMasterDataRequestService.createResultWithAttribute
        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, supplierMapper)
            ).thenReturn(mapped);

            // when
            List<CounterpartyDto> result = loader.fetchByKeys(keys);

            // then
            assertEquals(mapped, result);

            verify(criteriaBuilder).buildCriteriaFromKey(key);
            verify(properties).getSlugValueForCounterparty();
            verify(baseMasterDataRequestService).requestDataWithAttribute(slug, criteria);

            statics.verify(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, supplierMapper)
            );
        }
    }
}
```
