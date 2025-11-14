```java

@ExtendWith(MockitoExtension.class)
class LoaderTerBankByCodeTest {

    @Mock
    BaseMasterDataRequestService master;

    @Mock
    TerBankMapper mapper;

    @Mock
    SearchRequestProperties props;

    LoaderTerBankByCode loader;

    @BeforeEach
    void setUp() {
        loader = new LoaderTerBankByCode(master, mapper, props);
    }

    // -----------------------------------------------------------------------
    // cacheName / elementType / extractKey
    // -----------------------------------------------------------------------

    @Test
    void cacheName_returnsTerBankServiceConstant() {
        assertEquals(TerBankService2.TB_BY_CODE, loader.cacheName());
    }

    @Test
    void elementType_returnsTerBankDtoClass() {
        assertEquals(TerBankDto.class, loader.elementType());
    }

    @Test
    void extractKey_returnsTbCodeFromDto() {
        TerBankDto dto = mock(TerBankDto.class);
        when(dto.getTbCode()).thenReturn("TB001");

        String key = loader.extractKey(dto);

        assertEquals("TB001", key);
        verify(dto).getTbCode();
    }

    // -----------------------------------------------------------------------
    // Вариант А: "минимальный" тест fetchByKeys — только взаимодействие
    // -----------------------------------------------------------------------

    @Test
    void fetchByKeys_callsMasterWithSlugCodesAndBookContext() {
        List<String> codes = List.of("A", "B");
        when(props.getSlugValueForTerBank()).thenReturn("slug");

        // ВАЖНО: здесь мы не проверяем результат, только verify, что master вызван
        // Если createResult внутри требует не-null ответа, подставь здесь реальный тип.
        // TODO: замените Object на тип, который реально возвращает master.requestData(...)
        Object response = new Object();
        when(master.requestData("slug", codes, SearchRequestProperties.Context.BOOK))
                .thenReturn(response);

        // Результат нас не интересует — важно, что метод не падает и дергает master
        loader.fetchByKeys(codes);

        verify(props).getSlugValueForTerBank();
        verify(master).requestData("slug", codes, SearchRequestProperties.Context.BOOK);
    }

    // -----------------------------------------------------------------------
    // Вариант B: тест fetchByKeys с проверкой результата
    // (его нужно чуть адаптировать под твой проект)
    // -----------------------------------------------------------------------

    @Test
    void fetchByKeys_returnsResultFromCreateResult() {
        List<String> codes = List.of("A", "B");
        when(props.getSlugValueForTerBank()).thenReturn("slug");

        // TODO: замените РеальныйТипОтвета на тот тип, который возвращает master.requestData
        РеальныйТипОтвета response = mock(РеальныйТипОтвета.class);

        when(master.requestData("slug", codes, SearchRequestProperties.Context.BOOK))
                .thenReturn(response);

        // ожидаемый результат из createResult
        TerBankDto dto1 = new TerBankDto(); // или билдерами, как у тебя принято
        TerBankDto dto2 = new TerBankDto();
        List<TerBankDto> expected = List.of(dto1, dto2);

        // === Вариант 1: createResult — статический метод утилитного класса ===
        // Предположим, что он лежит в классе BaseMasterDataRequestServiceUtils:
        //
        // try (MockedStatic<BaseMasterDataRequestServiceUtils> utils =
        //          Mockito.mockStatic(BaseMasterDataRequestServiceUtils.class)) {
        //
        //     utils.when(() -> BaseMasterDataRequestServiceUtils
        //             .createResult(response, mapper))
        //           .thenReturn(expected);
        //
        //     List<TerBankDto> result = loader.fetchByKeys(codes);
        //     assertEquals(expected, result);
        // }
        //
        // === Вариант 2: createResult — protected / package-private метод базового класса
        // Тогда его можно замокать через spy на LoaderTerBankByCode.
        //
        // Например, если createResult определён в LoaderTerBankByCode как:
        // protected List<TerBankDto> createResult(РеальныйТипОтвета resp, TerBankMapper mapper)
        //
        // LoaderTerBankByCode spyLoader = spy(loader);
        // doReturn(expected).when(spyLoader).createResult(response, mapper);
        //
        // List<TerBankDto> result = spyLoader.fetchByKeys(codes);
        // assertEquals(expected, result);
        //
        // Здесь оставляю TODO, потому что точная сигнатура и расположение createResult
        // зависят от твоего кода.

        // TODO: вставь сюда один из вариантов выше и удали этот fail
        // fail("Доделай мокинг createResult в соответствии с реализацией.");
    }
}

```
