```java

@ExtendWith(MockitoExtension.class)
class NdsService2Test {

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    @InjectMocks
    private NdsService2 service;

    @Captor
    private ArgumentCaptor<List<String>> keysCaptor;

    /** Если rate = null/empty – формируется один ключ только с датой. */
    @Test
    @DisplayName("Когда rate не задан, в кеш уходит только ключ даты")
    void givenRateNull_whenGetBasicVatRate_thenOnlyDateKeyIsRequested() {
        // given
        ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");
        String expectedKey = NdsService2.DATE_TIME_FORMATTER.format(date);

        doAnswer(inv -> List.of())
                .when(cacheGetOrLoadService)
                .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), keysCaptor.capture());

        // when
        ResultObj<List<NdsDto>> result = service.getBasicVatRate(date, null, null);

        // then
        assertThat(result).isNotNull();
        assertThat(result.getData()).isEmpty();
        assertThat(keysCaptor.getValue()).containsExactly(expectedKey);

        verifyNoMoreInteractions(cacheGetOrLoadService);
    }

    /** Если передан список ставок – ключи содержат дату и ставку. */
    @Test
    @DisplayName("Если rate задан, в кеш уходят ключи вида <date>|<rate>")
    void givenRatesProvided_whenGetBasicVatRate_thenDateRateKeysAreRequested() {
        // given
        ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");
        String datePart = NdsService2.DATE_TIME_FORMATTER.format(date);

        List<String> rates = List.of("5", "10");

        doAnswer(inv -> List.of())
                .when(cacheGetOrLoadService)
                .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), keysCaptor.capture());

        // when
        service.getBasicVatRate(date, null, rates);

        // then
        assertThat(keysCaptor.getValue())
                .containsExactly(
                        datePart + "|5",
                        datePart + "|10"
                );
    }

    /** Проверка фильтрации по rate и code и маппинга в DTO. */
    @Test
    @DisplayName("Фильтры по rate и code корректно применяются к элементам из бакетов")
    void givenMixedItemsAndFilters_whenGetBasicVatRate_thenOnlyMatchedDtosReturned() {
        // given
        ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");
        String dateKey = NdsService2.DATE_TIME_FORMATTER.format(date);

        // pass1: пройдет всегда
        NdsFullDto pass1 = nds("1", "5", "A");
        // dropWrongRate: другая ставка
        NdsFullDto dropWrongRate = nds("2", "10", "B");
        // dropWrongCode: другой код
        NdsFullDto dropWrongCode = nds("3", "5", "C");
        // pass2: нужная ставка и код
        NdsFullDto pass2 = nds("4", "5", "B");

        List<NdsFullDto> all = List.of(pass1, dropWrongRate, dropWrongCode, pass2);

        List<NdsService2.RateBucket> buckets = List.of(
                new NdsService2.RateBucket(dateKey, Set.copyOf(all))
        );

        // любые ключи -> возвращаем заранее подготовленные бакеты
        doAnswer(inv -> buckets)
                .when(cacheGetOrLoadService)
                .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), anyList());

        // when
        List<NdsDto> r1 = service.getBasicVatRate(date, null, null).getData();                // без фильтров
        List<NdsDto> r2 = service.getBasicVatRate(date, null, List.of("5")).getData();       // фильтр только по rate
        List<NdsDto> r3 = service.getBasicVatRate(date, List.of("B"), null).getData();       // фильтр только по code
        List<NdsDto> r4 = service.getBasicVatRate(date, List.of("B"), List.of("5")).getData(); // оба фильтра

        // then
        assertThat(r1).extracting(NdsDto::getId)
                .containsExactlyInAnyOrder("1", "2", "3", "4");

        assertThat(r2).extracting(NdsDto::getId)
                .containsExactlyInAnyOrder("1", "3", "4");   // только rate=5

        assertThat(r3).extracting(NdsDto::getId)
                .containsExactlyInAnyOrder("2", "4");        // только code=B

        assertThat(r4).extracting(NdsDto::getId)
                .containsExactly("4");                       // rate=5 и code=B

        // Дополнительно проверим, что поля у прошедшего DTO корректные
        NdsDto dto = r4.get(0);
        assertThat(dto.getRate()).isEqualTo("5");
        assertThat(dto.getCode()).isEqualTo("B");
        assertThat(dto.getName()).isEqualTo("N4");
    }

    @Test
@DisplayName("Если из кеша приходит пустой список бакетов, то результат пустой")
void givenEmptyBuckets_whenGetBasicVatRate_thenEmptyResult() {
    doAnswer(inv -> List.of())
            .when(cacheGetOrLoadService)
            .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), anyList());

    var result = service.getBasicVatRate(ZonedDateTime.now(), null, null);

    assertThat(result.getData()).isEmpty();
}

@Test
@DisplayName("Если rate пустой список, ключ кеша тот же, что и при null")
void givenEmptyRateList_whenGetBasicVatRate_thenOnlyDateKeyRequested() {
    ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");
    String expectedKey = NdsService2.DATE_TIME_FORMATTER.format(date);

    doAnswer(inv -> List.of())
            .when(cacheGetOrLoadService)
            .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), keysCaptor.capture());

    service.getBasicVatRate(date, null, List.of());

    assertThat(keysCaptor.getValue()).containsExactly(expectedKey);
}

@Test
@DisplayName("buildKeys фильтрует null/пустые значения rate, но не падает")
void givenDirtyRates_whenGetBasicVatRate_thenKeyBuiltOnlyForValidRates() {
    ZonedDateTime date = ZonedDateTime.parse("2025-07-21T10:00:00+03:00");
    String datePart = NdsService2.DATE_TIME_FORMATTER.format(date);

    List<String> rates = List.of(null, "  ", "5");

    doAnswer(inv -> List.of())
            .when(cacheGetOrLoadService)
            .fetchData(eq(NdsService2.CACHE_NDS_BY_RATE), keysCaptor.capture());

    service.getBasicVatRate(date, null, rates);

    assertThat(keysCaptor.getValue()).containsExactly(datePart + "|5");
}

    /** Упрощённый билдер тестовых NdsFullDto. */
    private static NdsFullDto nds(String id, String rate, String code) {
        return NdsFullDto.builder()
                .id(id)
                .name("N" + id)
                .rate(rate)
                .code(code)
                .build();
    }
}
```
