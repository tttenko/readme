```java

/**
 * Юнит-тесты для CacheGetOrLoadService.
 * Проверяем только взаимодействие с BatchCacheSupport и BatchLoader.
 */
@ExtendWith(MockitoExtension.class)
class CacheGetOrLoadServiceTest {

    private static final String CACHE_NAME = "tb_by_code";
    private static final String UNKNOWN_CACHE = "unknown_cache";

    @Mock
    BatchCacheSupport cache;

    @Mock
    BatchLoader<TestDto> loader;

    CacheGetOrLoadService service;

    @BeforeEach
    void setUp() {
        // Лоадер привязываем к нашему cacheName
        when(loader.cacheName()).thenReturn(CACHE_NAME);
        when(loader.elementType()).thenReturn(TestDto.class);

        // Конструируем сервис с одним лоадером
        service = new CacheGetOrLoadService(cache, List.of(loader));

        // Эмулируем вызов @PostConstruct
        service.init();

        // Чистим историю вызовов, чтобы init не мешал verify в тестах
        clearInvocations(cache, loader);
    }

    // ========================================================================
    // 1. Пустой список ключей
    // ========================================================================

    @Test
    void fetchData_emptyKeys_returnsEmptyAndDoesNotUseCacheOrLoader() {
        List<TestDto> result = service.fetchData(CACHE_NAME, List.of());

        assertNotNull(result);
        assertTrue(result.isEmpty());

        // Никаких обращений к кэшу и лоадеру не должно быть
        verifyNoInteractions(cache);
        verifyNoInteractions(loader);
    }

    // ========================================================================
    // 2. Лоадер для cacheName не найден
    // ========================================================================

    @Test
    void fetchData_loaderNotFound_throwsBatchLoadExceptionAndDoesNotUseCache() {
        List<String> keys = List.of("A");

        MdaBatchLoadException ex = assertThrows(
                MdaBatchLoadException.class,
                () -> service.fetchData(UNKNOWN_CACHE, keys)
        );
        assertTrue(ex.getMessage().contains(UNKNOWN_CACHE));

        // Кэш не должен трогаться
        verifyNoInteractions(cache);
        // И лоадер тоже — мы его вообще не достали из byCache
        verifyNoInteractions(loader);
    }

    // ========================================================================
    // 3. Все ключи уже есть в кэше (miss пустой)
    // ========================================================================

    @Test
    void fetchData_allKeysCached_loaderNotCalled_resultFromCache() {
        List<String> keys = List.of("A", "B");

        List<TestDto> cached = List.of(
                new TestDto("A"),
                new TestDto("B")
        );

        // Промахов нет
        when(cache.collectMisses(CACHE_NAME, keys, TestDto.class))
                .thenReturn(List.of());

        // Читаем из кэша готовые значения
        when(cache.readFromCache(CACHE_NAME, keys, TestDto.class))
                .thenReturn(cached);

        List<TestDto> result = service.fetchData(CACHE_NAME, keys);

        assertEquals(cached, result);

        // Проверяем взаимодействия
        verify(cache).collectMisses(CACHE_NAME, keys, TestDto.class);
        verify(cache).readFromCache(CACHE_NAME, keys, TestDto.class);

        // loader.fetchByKeys и cache.putToCache вызываться не должны
        verify(loader, never()).fetchByKeys(anyList());
        verify(cache, never()).putToCache(anyString(), anyList(), any());

        // elementType() вызывается, т.к. он нужен для collectMisses/readFromCache
        verify(loader, times(2)).elementType();
    }

    // ========================================================================
    // 4. Есть промахи: loader вызывается только по miss-ам, результат кладётся в кэш
    // ========================================================================

    @Test
    void fetchData_missesPresent_loaderCalledForMisses_andCacheUpdatedAndRead() {
        List<String> keys = List.of("A", "B", "C");
        List<String> miss = List.of("B", "C");

        TestDto dtoB = new TestDto("B");
        TestDto dtoC = new TestDto("C");
        List<TestDto> loaded = List.of(dtoB, dtoC);

        // Промахи по двум ключам
        when(cache.collectMisses(CACHE_NAME, keys, TestDto.class))
                .thenReturn(miss);

        // Лоадер грузит только промахи
        when(loader.fetchByKeys(miss)).thenReturn(loaded);

        // extractKey просто возвращает code — нам так удобно проверить функцию
        when(loader.extractKey(any())).thenAnswer(invocation ->
                ((TestDto) invocation.getArgument(0)).code()
        );

        // После того, как всё положили в кэш, сервис читает из кэша итоговый список
        List<TestDto> fromCache = List.of(
                new TestDto("A"),
                dtoB,
                dtoC
        );
        when(cache.readFromCache(CACHE_NAME, keys, TestDto.class))
                .thenReturn(fromCache);

        List<TestDto> result = service.fetchData(CACHE_NAME, keys);

        // Сервис должен вернуть то, что вернул readFromCache
        assertEquals(fromCache, result);

        // --- Проверяем, что loader и cache вызваны правильно ---

        // Промахи собрали один раз
        verify(cache).collectMisses(CACHE_NAME, keys, TestDto.class);

        // Лоадер вызван ровно по этому списку miss
        verify(loader).fetchByKeys(miss);

        // Захватываем аргументы putToCache
        @SuppressWarnings("unchecked")
        ArgumentCaptor<List<TestDto>> loadedCaptor =
                ArgumentCaptor.forClass(List.class);
        @SuppressWarnings("unchecked")
        ArgumentCaptor<Function<TestDto, String>> keyFnCaptor =
                ArgumentCaptor.forClass(Function.class);

        verify(cache).putToCache(eq(CACHE_NAME), loadedCaptor.capture(), keyFnCaptor.capture());

        // В кэш положили именно тот список, который вернул loader
        assertSame(loaded, loadedCaptor.getValue());

        // Функция ключа действительно делегирует в loader.extractKey
        Function<TestDto, String> keyFn = keyFnCaptor.getValue();
        String keyB = keyFn.apply(dtoB);
        assertEquals("B", keyB);
        verify(loader).extractKey(dtoB);

        // Затем значения читаются из кэша
        verify(cache).readFromCache(CACHE_NAME, keys, TestDto.class);

        // elementType() вызван два раза — для collectMisses и readFromCache
        verify(loader, times(2)).elementType();
    }

    // ========================================================================
    // 5. Есть промахи, но loader вернул пустой список
    //    (крайний случай — просто проверим, что putToCache всё равно вызывается)
    // ========================================================================

    @Test
    void fetchData_missesPresent_butLoaderReturnsEmpty_stillPutsAndReadsFromCache() {
        List<String> keys = List.of("A", "B");
        List<String> miss = List.of("B");

        when(cache.collectMisses(CACHE_NAME, keys, TestDto.class))
                .thenReturn(miss);

        // Лоадер ничего не нашёл
        List<TestDto> loaded = List.of();
        when(loader.fetchByKeys(miss)).thenReturn(loaded);

        when(cache.readFromCache(CACHE_NAME, keys, TestDto.class))
                .thenReturn(List.of(new TestDto("A")));

        List<TestDto> result = service.fetchData(CACHE_NAME, keys);

        assertEquals(1, result.size());
        assertEquals("A", result.get(0).code());

        verify(loader).fetchByKeys(miss);
        verify(cache).putToCache(eq(CACHE_NAME), eq(loaded), any());
        verify(cache).readFromCache(CACHE_NAME, keys, TestDto.class);
    }

    // ------------------------------------------------------------------------
    // Вспомогательный DTO для тестов
    // ------------------------------------------------------------------------

    /**
     * Простой иммутабельный объект, чтобы не подтягивать реальные сущности.
     */
    private record TestDto(String code) {
    }
}

```
