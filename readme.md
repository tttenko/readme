```java
@SpringBootTest(classes = {
        NdsUiControllerImpl.class,
        SearchRequestProperties.class,
        HttpRequestHelper.class,
        NdsService.class,
        NdsMapper.class,
        GlobalExceptionHandler.class,
        CacheGetOrLoadService.class,
        BatchCacheSupport.class,
        LoaderNdsByRate.class,
        CacheConfig.class,
        BaseMasterDataRequestService.class,
        ResponseHandler.class
})
class NdsUiControllerMvcTest {

    private AutoCloseable closeable;

    @MockitoBean
    private ObjectMapper mapper;

    @Autowired
    private NdsService service;

    @Autowired
    private ResponseHandler responseHandler;

    @MockitoBean
    private HttpRequestHelper httpRequestHelper;

    @MockitoBean
    private SearchRequestProperties searchRequestProperties;

    private MockMvc mockMvc;
    private ThreadSafeResourceReader reader;

    @BeforeEach
    void setUp() {
        // важно: контроллер создаём руками как в твоих тестах
        UiNdsController controller = new NdsUiControllerImpl(service, responseHandler);

        reader = MvcTestUtils.createReader(this);
        closeable = MockitoAnnotations.openMocks(this);

        // важно: чтобы мок совпал с реальным URL, который возьмёт сервис
        when(searchRequestProperties.getGetListByAttrValuesUri(SearchRequestProperties.Context.BOOK))
                .thenReturn("v1/items/byAttrValues");

        // если в твоём SearchRequestProperties есть методы для NDS (slug/attrId) и они реально используются,
        // и без них уходит некорректный запрос — добавь аналогично:
        // when(searchRequestProperties.getSlugValueForNds()).thenReturn("nds");
        // when(searchRequestProperties.getAttributeIdForNds()).thenReturn("...");

        this.mockMvc = MockMvcBuilders
                .standaloneSetup(controller)
                .defaultResponseCharacterEncoding(StandardCharsets.UTF_8)
                .build();

        // если mapper реально мок — эта настройка ни на что не влияет, но оставляем как в твоих тестах
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }

    @AfterEach
    void tearDown() throws Exception {
        closeable.close();
    }

    @Test
    @DisplayName("test GET {host}/ui/v1/main-nds")
    void searchUiNdsAllTest() throws Exception {
        GetItemsSearchResponse response = reader.readResource(
                "mdresponse/nds/nds-type-1.json",
                GetItemsSearchResponse.class
        );

        MvcTestUtils.mockPostResponse(
                "v1/items/byAttrValues",
                response,
                GetItemsSearchResponse.class,
                httpRequestHelper
        );

        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds"),
                6
        );
    }

    @Test
    @DisplayName("test GET {host}/ui/v1/main-nds (cache + params)")
    void searchUiNdsCacheTest() throws Exception {
        GetItemsSearchResponse response = reader.readResource(
                "mdresponse/nds/nds-type-1.json",
                GetItemsSearchResponse.class
        );

        MvcTestUtils.mockPostResponse(
                "v1/items/byAttrValues",
                response,
                GetItemsSearchResponse.class,
                httpRequestHelper
        );

        // разные фильтры
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds?rate=0"), 2);
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds?rate=5"), 1);

        // чистим кэш (как в твоём примере)
        service.cleanCache();

        // повторный вызов после чистки
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds?rate=5"), 1);
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds"), 6);

        // дата + таймзона: "+" обязательно кодируем как %2B
        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc,
                        "/ui/v1/main-nds?rate=5&date=2025-07-21T10:00:00%2B03:00"),
                1
        );
    }

    @Test
    @DisplayName("test GET {host}/ui/v1/main-nds-code")
    void searchUiNdsCodeCacheTest() throws Exception {
        GetItemsSearchResponse response = reader.readResource(
                "mdresponse/nds/nds-type-1.json",
                GetItemsSearchResponse.class
        );

        MvcTestUtils.mockPostResponse(
                "v1/items/byAttrValues",
                response,
                GetItemsSearchResponse.class,
                httpRequestHelper
        );

        String date = "2025-07-21T10:00:00%2B03:00";

        // без code/rate
        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds-code?date=" + date),
                6
        );

        // с code
        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds-code?code=NOVAT&date=" + date),
                1
        );

        // с code + rate
        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds-code?code=NOVAT&rate=0&date=" + date),
                1
        );

        // чистим кэш и повторяем
        service.cleanCache();

        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds-code?code=NOVAT&rate=0&date=" + date),
                1
        );
    }
}
```
