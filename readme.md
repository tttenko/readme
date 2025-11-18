```java

 @ExtendWith(MockitoExtension.class)
class LoaderNdsByRateTest {

    private static final String TYPE_ID   = "00000000-0000-0000-0000-000000000001";
    private static final String ACTIVE_ID = "00000000-0000-0000-0000-000000000002";
    private static final String END_ID    = "00000000-0000-0000-0000-000000000003"; // больше не используются, но мокаем для надёжности
    private static final String START_ID  = "00000000-0000-0000-0000-000000000004";
    private static final String ALL_KEY   = "__ALL__";

    @Mock
    private BaseMasterDataRequestService baseService;

    @Mock
    private NdsMapper ndsMapper;

    @Mock
    private SearchRequestProperties properties;

    @InjectMocks
    private LoaderNdsByRate loader;

    @Captor
    private ArgumentCaptor<ItemsSearchCriteriaRequest> reqCaptor;

    @BeforeEach
    void setUp() {
        lenient().when(properties.getSlugValueForVat()).thenReturn("vat");
        lenient().when(properties.getAttributeIdForTaxRateType()).thenReturn(TYPE_ID);
        lenient().when(properties.getAttributeIdForTaxRateActive()).thenReturn(ACTIVE_ID);
        // эти два сейчас не используются в buildRequest, но могут пригодиться другим тестам
        lenient().when(properties.getAttributeIdForTaxRateEndDate()).thenReturn(END_ID);
        lenient().when(properties.getAttributeIdForTaxRateStartDate()).thenReturn(START_ID);
    }

    @Test
    void givenEmptyKeys_whenFetch_thenEmptyAndNoCalls() {
        List<NdsService2.RateBucket> buckets = loader.fetchByKeys(List.of());

        assertThat(buckets).isEmpty();
        verifyNoInteractions(baseService, ndsMapper);
    }

    @Test
    void givenSingleCodeKey_whenFetch_thenSingleBucketWithThatCode() {
        // ключ кеша — просто код налога
        String key = "A";

        var itemA = NdsFullDto.builder()
                .id("1").name("N1").rate("5").code("A")
                .build();
        var itemB = NdsFullDto.builder()
                .id("2").name("N2").rate("10").code("B")
                .build();

        var items = List.of(itemA, itemB);
        GetItemsSearchResponse response = new GetItemsSearchResponse();

        try (MockedStatic<BaseMasterDataRequestService> st =
                     mockStatic(BaseMasterDataRequestService.class)) {

            when(baseService.requestData(any(ItemsSearchCriteriaRequest.class),
                    eq(SearchRequestProperties.Context.BOOK)))
                    .thenReturn(response);

            st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(response, ndsMapper))
                    .thenReturn(items);

            // when
            List<NdsService2.RateBucket> buckets = loader.fetchByKeys(List.of(key));

            // then
            assertThat(buckets).hasSize(1);
            NdsService2.RateBucket bucket = buckets.get(0);

            assertThat(bucket.key()).isEqualTo(key);
            assertThat(bucket.items())
                    .extracting(NdsFullDto::getId)
                    .containsExactly("1"); // только элементы с code = "A"

            verify(baseService, times(1))
                    .requestData(any(ItemsSearchCriteriaRequest.class),
                            eq(SearchRequestProperties.Context.BOOK));
        }
    }

    @Test
    void givenALLKey_whenFetch_thenUnionBucket() {
        String allKey = ALL_KEY;
        String keyA = "A";
        String keyB = "B";

        var itemA = NdsFullDto.builder()
                .id("1").name("N1").rate("5").code("A")
                .build();
        var itemB = NdsFullDto.builder()
                .id("2").name("N2").rate("10").code("B")
                .build();

        var items = List.of(itemA, itemB);
        GetItemsSearchResponse response = new GetItemsSearchResponse();

        try (MockedStatic<BaseMasterDataRequestService> st =
                     mockStatic(BaseMasterDataRequestService.class)) {

            when(baseService.requestData(any(ItemsSearchCriteriaRequest.class),
                    eq(SearchRequestProperties.Context.BOOK)))
                    .thenReturn(response);

            st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(response, ndsMapper))
                    .thenReturn(items);

            // when
            List<NdsService2.RateBucket> buckets =
                    loader.fetchByKeys(List.of(allKey, keyA, keyB));

            // then
            NdsService2.RateBucket allBucket = findBucket(buckets, allKey);
            NdsService2.RateBucket bucketA = findBucket(buckets, keyA);
            NdsService2.RateBucket bucketB = findBucket(buckets, keyB);

            // __ALL__ — объединение всех элементов
            assertThat(allBucket.items())
                    .extracting(NdsFullDto::getId)
                    .containsExactlyInAnyOrder("1", "2");

            assertThat(bucketA.items())
                    .extracting(NdsFullDto::getId)
                    .containsExactly("1");

            assertThat(bucketB.items())
                    .extracting(NdsFullDto::getId)
                    .containsExactly("2");

            verify(baseService, times(1))
                    .requestData(any(ItemsSearchCriteriaRequest.class),
                            eq(SearchRequestProperties.Context.BOOK));
        }
    }

    @Test
    void givenUnknownCodeKey_whenFetch_thenEmptyBucket() {
        String keyUnknown = "UNKNOWN";

        var items = List.of(
                NdsFullDto.builder()
                        .id("1").name("N1").rate("5").code("A")
                        .build()
        );
        GetItemsSearchResponse response = new GetItemsSearchResponse();

        try (MockedStatic<BaseMasterDataRequestService> st =
                     mockStatic(BaseMasterDataRequestService.class)) {

            when(baseService.requestData(any(ItemsSearchCriteriaRequest.class),
                    eq(SearchRequestProperties.Context.BOOK)))
                    .thenReturn(response);

            st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(response, ndsMapper))
                    .thenReturn(items);

            // when
            List<NdsService2.RateBucket> buckets = loader.fetchByKeys(List.of(keyUnknown));

            // then
            assertThat(buckets).hasSize(1);
            NdsService2.RateBucket bucket = buckets.get(0);

            assertThat(bucket.key()).isEqualTo(keyUnknown);
            assertThat(bucket.items()).isEmpty(); // code "UNKNOWN" ни к чему не мапится
        }
    }

    @Test
    void givenProps_whenFetch_thenDictionaryAndFiltersApplied() {
        String key = "A";

        GetItemsSearchResponse response = new GetItemsSearchResponse();

        try (MockedStatic<BaseMasterDataRequestService> st =
                     mockStatic(BaseMasterDataRequestService.class)) {

            when(baseService.requestData(reqCaptor.capture(),
                    eq(SearchRequestProperties.Context.BOOK)))
                    .thenReturn(response);

            st.when(() -> BaseMasterDataRequestService.createResultWithAttribute(response, ndsMapper))
                    .thenReturn(List.of());

            // when
            loader.fetchByKeys(List.of(key));

            // then
            ItemsSearchCriteriaRequest req = reqCaptor.getValue();
            assertThat(req).isNotNull();
            // проверяем, что подставился нужный словарь
            assertThat(req.getReference().getSlug()).isEqualTo("vat");

            // и что в запросе используются нужные атрибуты (тип + активность)
            String asString = req.toString();
            assertThat(asString).contains(TYPE_ID);
            assertThat(asString).contains(ACTIVE_ID);
            // а даты там больше нет
            assertThat(asString).doesNotContain("2025-07-21");
        }
    }

    private static NdsService2.RateBucket findBucket(List<NdsService2.RateBucket> buckets,
                                                     String key) {
        return buckets.stream()
                .filter(b -> b.key().equals(key))
                .findFirst()
                .orElseThrow();
    }
}
        

```
