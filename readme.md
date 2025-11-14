```java

@ExtendWith(MockitoExtension.class)
class LoaderSupplierByIdTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SupplierMapper supplierMapper;

    @Mock
    private SearchRequestProperties properties;

    @InjectMocks
    private LoaderSupplierById loader;

    // -------------------------------------------------------
    // Простые методы: cacheName / elementType / extractKey
    // -------------------------------------------------------

    @Test
    void givenLoader_whenCacheNameCalled_thenReturnSupplierByIdConstant() {
        // when
        String name = loader.cacheName();

        // then
        assertEquals(SupplierService2.SUPPLIER_BY_ID, name);
    }

    @Test
    void givenLoader_whenElementTypeCalled_thenReturnCounterpartyDtoClass() {
        // when
        Class<CounterpartyDto> type = loader.elementType();

        // then
        assertEquals(CounterpartyDto.class, type);
    }

    @Test
    void givenDto_whenExtractKeyCalled_thenReturnDtoId() {
        // given
        CounterpartyDto dto = mock(CounterpartyDto.class);
        when(dto.getId()).thenReturn("ID1");

        // when
        String key = loader.extractKey(dto);

        // then
        assertEquals("ID1", key);
        verify(dto).getId();
    }

    // -------------------------------------------------------
    // fetchByKeys
    // -------------------------------------------------------

    @Test
    void givenEmptyIds_whenFetchByKeys_thenReturnEmptyAndDoNotCallBaseService() {
        // when
        List<CounterpartyDto> result = loader.fetchByKeys(List.of());

        // then
        assertNotNull(result);
        assertTrue(result.isEmpty());

        // ни одного вызова сервисов
        verifyNoInteractions(baseMasterDataRequestService, properties, supplierMapper);
    }

    @Test
    void givenIds_whenFetchByKeys_thenCallBaseServiceAndMapResult() {
        // given
        List<String> ids = List.of("A", "B");
        String slug = "counterpartySlug";

        when(properties.getSlugValueForCounterparty()).thenReturn(slug);

        GetItemsSearchResponse resp = mock(GetItemsSearchResponse.class);
        when(baseMasterDataRequestService.requestDataWithAttribute(
                slug,
                ids,
                SearchRequestProperties.Context.BOOK
        )).thenReturn(resp);

        List<CounterpartyDto> mapped = List.of(
                mock(CounterpartyDto.class),
                mock(CounterpartyDto.class)
        );

        // мок статического метода BaseMasterDataRequestService.createResultWithAttribute
        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class, CALLS_REAL_METHODS)) {

            statics.when(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, supplierMapper)
            ).thenReturn(mapped);

            // when
            List<CounterpartyDto> result = loader.fetchByKeys(ids);

            // then
            assertEquals(mapped, result);

            verify(properties).getSlugValueForCounterparty();
            verify(baseMasterDataRequestService).requestDataWithAttribute(
                    slug, ids, SearchRequestProperties.Context.BOOK
            );

            statics.verify(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, supplierMapper)
            );
        }
    }
}
```
