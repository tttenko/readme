```java

@ExtendWith(MockitoExtension.class)
class NdsService2Test {

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    @InjectMocks
    private NdsService2 service;

    @Captor
    private ArgumentCaptor<List<String>> keysCaptor;

    private static final DateTimeFormatter DATE_TIME_FORMATTER =
            DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mmXXX");

    /**
     * Если не переданы ни code, ни rate – NdsService2 должен запросить из кеша
     * «полный» бакет по специальному ключу __ALL__.
     */
    @Test
    void givenCodeAndRateNull_whenGetBasicVatRate_thenAllKeyIsRequested() {
        // given
        ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");

        doAnswer(inv -> List.of())
                .when(cacheGetOrLoadService)
                .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), keysCaptor.capture());

        // when
        ResultObj<List<NdsDto>> result = service.getBasicVatRate(date, null, null);

        // then
        assertThat(result).isNotNull();
        assertThat(result.getData()).isEmpty();
        assertThat(keysCaptor.getValue()).containsExactly("__ALL__");

        verifyNoMoreInteractions(cacheGetOrLoadService);
    }

    /**
     * Теперь rate никак не влияет на ключ кеша: даже если список ставок передан,
     * но кодов нет, всё равно должен запрашиваться только общий бакет __ALL__.
     */
    @Test
    void givenRatesProvidedWithoutCodes_whenGetBasicVatRate_thenStillAllKeyIsRequested() {
        // given
        ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");
        List<String> rates = List.of("5", "10");

        doAnswer(inv -> List.of())
                .when(cacheGetOrLoadService)
                .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), keysCaptor.capture());

        // when
        service.getBasicVatRate(date, null, rates);

        // then – ключ кеша такой же, как и без ставок
        assertThat(keysCaptor.getValue()).containsExactly("__ALL__");
    }

    /**
     * Если переданы коды, именно они используются как ключи кеша
     * (после тримминга и удаления дублей/пустых значений).
     */
    @Test
    void givenCodesProvided_whenGetBasicVatRate_thenCodeKeysAreRequested() {
        // given
        ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");
        List<String> codes = List.of(" A ", "B");

        doAnswer(inv -> List.of())
                .when(cacheGetOrLoadService)
                .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), keysCaptor.capture());

        // when
        service.getBasicVatRate(date, codes, null);

        // then
        assertThat(keysCaptor.getValue()).containsExactly("A", "B");
    }

    /**
     * Поведение фильтрации по rate/code (и активным датам) остаётся прежним:
     * здесь мы не проверяем ключ кеша, а только то, какие DTO вернулись.
     */
    @Test
    void givenMixedItemsAndFilters_whenGetBasicVatRate_thenOnlyMatchedDtosReturned() {
        // given
        ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");
        String anyBucketKey = "any"; // ключ нам здесь не важен

        NdsFullDto pass1         = nds("1", "5",  "A");
        NdsFullDto dropWrongRate = nds("2", "10", "B");
        NdsFullDto dropWrongCode = nds("3", "5",  "C");
        NdsFullDto pass2         = nds("4", "5",  "B");

        List<NdsFullDto> all = List.of(pass1, dropWrongRate, dropWrongCode, pass2);

        List<NdsService2.RateBucket> buckets = List.of(
                new NdsService2.RateBucket(anyBucketKey, Set.copyOf(all))
        );

        doAnswer(inv -> buckets)
                .when(cacheGetOrLoadService)
                .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), anyList());

        // when
        List<NdsDto> r1 = service.getBasicVatRate(date, null, null).getData();
        List<NdsDto> r2 = service.getBasicVatRate(date, null, List.of("5")).getData();
        List<NdsDto> r3 = service.getBasicVatRate(date, List.of("B"), null).getData();
        List<NdsDto> r4 = service.getBasicVatRate(date, List.of("B"), List.of("5")).getData();

        // then
        assertThat(r1).extracting(NdsDto::getId)
                .containsExactlyInAnyOrder("1", "2", "3", "4");

        assertThat(r2).extracting(NdsDto::getId)
                .containsExactlyInAnyOrder("1", "3", "4");

        assertThat(r3).extracting(NdsDto::getId)
                .containsExactlyInAnyOrder("2", "4");

        assertThat(r4).extracting(NdsDto::getId)
                .containsExactly("4");

        NdsDto dto = r4.get(0);
        assertThat(dto.getRate()).isEqualTo("5");
        assertThat(dto.getCode()).isEqualTo("B");
        assertThat(dto.getName()).isEqualTo("N4");
    }

    /**
     * Если из кеша пришёл пустой список бакетов – результат тоже пустой.
     */
    @Test
    void givenEmptyBuckets_whenGetBasicVatRate_thenEmptyResult() {
        doAnswer(inv -> List.of())
                .when(cacheGetOrLoadService)
                .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), anyList());

        var result = service.getBasicVatRate(ZonedDateTime.now(), null, null);

        assertThat(result.getData()).isEmpty();
    }

    /**
     * Пустой список code должен обрабатываться так же, как и null:
     * запрашиваем общий бакет __ALL__.
     */
    @Test
    void givenEmptyCodeList_whenGetBasicVatRate_thenAllKeyIsRequested() {
        ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");

        doAnswer(inv -> List.of())
                .when(cacheGetOrLoadService)
                .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), keysCaptor.capture());

        service.getBasicVatRate(date, List.of(), null);

        assertThat(keysCaptor.getValue()).containsExactly("__ALL__");
    }

    /**
     * "Грязные" коды (null/пробелы/дубли) должны быть отфильтрованы,
     * в ключи кеша попадает только нормализованный набор кодов.
     */
    @Test
    void givenDirtyCodes_whenGetBasicVatRate_thenKeyBuiltOnlyForValidCodes() {
        ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");

        List<String> codes = Arrays.asList(null, " ", "A", "A");

        doAnswer(inv -> List.of())
                .when(cacheGetOrLoadService)
                .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), keysCaptor.capture());

        service.getBasicVatRate(date, codes, null);

        // остаётся только один чистый код "A"
        assertThat(keysCaptor.getValue()).containsExactly("A");
    }

    // ----------------------------------------------------------------------
    // helpers
    // ----------------------------------------------------------------------
    private static NdsFullDto nds(String id, String rate, String code) {
        return NdsFullDto.builder()
                .id(id)
                .name("N" + id)
                .rate(rate)
                .code(code)
                // даты не задаём: по новой логике это значит "ставка всегда активна"
                .build();
    }
}

```
