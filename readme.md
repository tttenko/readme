```java

@ExtendWith(MockitoExtension.class)
class LoaderNdsByRateTest {

    private static final String TYPE_ID  = "00000000-0000-0000-0000-000000000001";
    private static final String ACTIVE_ID = "00000000-0000-0000-0000-000000000002";
    private static final String END_ID    = "00000000-0000-0000-0000-000000000003";
    private static final String START_ID  = "00000000-0000-0000-0000-000000000004";

    @Mock
    private BaseMasterDataRequestService baseService;

    @Mock
    private NdsMapper ndsMapper;

    @Mock
    private SearchRequestProperties properties;

    @InjectMocks
    private LoaderNdsByRate loader;   // тестируем ровно этот класс

    @BeforeEach
    void setUp() {
        // настройки из старого теста – чтобы билдился правильный запрос
        lenient().when(properties.getSlugValueForVat()).thenReturn("vat");
        lenient().when(properties.getAttributeIdForTaxRateType()).thenReturn(TYPE_ID);
        lenient().when(properties.getAttributeIdForTaxRateActive()).thenReturn(ACTIVE_ID);
        lenient().when(properties.getAttributeIdForTaxRateEndDate()).thenReturn(END_ID);
        lenient().when(properties.getAttributeIdForTaxRateStartDate()).thenReturn(START_ID);
    }

    /** Пустой список ключей – ничего не грузим, МД не дергаем. */
    @Test
    void givenEmptyKeys_whenFetch_thenEmptyAndNoCalls() {
        List<NdsService2.RateBucket> buckets = loader.fetchByKeys(List.of());

        assertThat(buckets).isEmpty();
        verifyNoInteractions(baseService, ndsMapper);
    }

    /** Один ключ с конкретной ставкой -> один бакет с элементами этой ставки. */
    @Test
    void givenSingleRateKey_whenFetch_thenSingleBucketWithThatRate() {
        String dateString = "2025-07-21T10:00:00+03:00";
        String key = dateString + "|5";

        ZonedDateTime date = ZonedDateTime.parse(dateString);

        var items = List.of(
                NdsFullDto.builder()
                        .id("1").name("N1").rate("5").code("A")
                        .rateDateStartZoned(date.minusDays(1))
                        .rateDateEndZoned(null)
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
            List<NdsService2.RateBucket> buckets = loader.fetchByKeys(List.of(key));

            // then
            assertThat(buckets).hasSize(1);
            NdsService2.RateBucket bucket = buckets.get(0);

            assertThat(bucket.key()).isEqualTo(key);
            assertThat(bucket.items())
                    .extracting(NdsFullDto::getId)
                    .containsExactly("1");

            verify(baseService, times(1))
                    .requestData(any(ItemsSearchCriteriaRequest.class),
                            eq(SearchRequestProperties.Context.BOOK));
        }
    }

    /** Ключ без ставки (только дата) -> "общий" бакет с объединением всех элементов. */
    @Test
    void givenAllKey_whenFetch_thenUnionBucket() {
        String dateString = "2025-07-21T10:00:00+03:00";
        String allKey = dateString;          // без '|'
        String key5  = dateString + "|5";
        String key10 = dateString + "|10";

        ZonedDateTime date = ZonedDateTime.parse(dateString);

        var item5 = NdsFullDto.builder()
                .id("1").name("N1").rate("5").code("A")
                .rateDateStartZoned(date.minusDays(5))
                .build();

        var item10 = NdsFullDto.builder()
                .id("2").name("N2").rate("10").code("B")
                .rateDateStartZoned(date.minusDays(5))
                .build();

        var items = List.of(item5, item10);
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
                    loader.fetchByKeys(List.of(allKey, key5, key10));

            // then
            NdsService2.RateBucket allBucket  = findBucket(buckets, allKey);
            NdsService2.RateBucket bucket5   = findBucket(buckets, key5);
            NdsService2.RateBucket bucket10  = findBucket(buckets, key10);

            // общий бакет – объединение всех элементов
            assertThat(allBucket.items())
                    .extracting(NdsFullDto::getId)
                    .containsExactlyInAnyOrder("1", "2");

            // бакеты по ставкам – только свои элементы
            assertThat(bucket5.items())
                    .extracting(NdsFullDto::getId)
                    .containsExactly("1");
            assertThat(bucket10.items())
                    .extracting(NdsFullDto::getId)
                    .containsExactly("2");
        }
    }

    /** Ключ c ставкой, которой нет в МД -> пустой бакет. */
    @Test
    void givenUnknownRateKey_whenFetch_thenEmptyBucket() {
        String dateString = "2025-07-21T10:00:00+03:00";
        String keyUnknown = dateString + "|99";

        ZonedDateTime date = ZonedDateTime.parse(dateString);

        var items = List.of(
                NdsFullDto.builder()
                        .id("1").name("N1").rate("5").code("A")
                        .rateDateStartZoned(date.minusDays(5))
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
            assertThat(bucket.items()).isEmpty();   // для неизвестной ставки – пустой набор
        }
    }

    private static NdsService2.RateBucket findBucket(List<NdsService2.RateBucket> buckets, String key) {
        return buckets.stream()
                .filter(b -> b.key().equals(key))
                .findFirst()
                .orElseThrow();
    }
}

```
