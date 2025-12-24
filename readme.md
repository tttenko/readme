```java
@SpringBootTest(classes = {
        // 👇 замени на свой UI-контроллер-имплементацию
        NdsUiControllerImpl.class,

        // 👇 остальной список оставь как в твоём NdsControllerMvcTest
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
@ActiveProfiles("test")
class NdsUiControllerMvcTest {

    // ====== ПОДСТАВЬ СВОИ UI PATH ======
    private static final String UI_V1 = "/ui/v1";              // например "/ui/v1" или "/ui/v1/info" — как у тебя заведено
    private static final String MAIN_NDS = UI_V1 + "/main-nds";
    private static final String MAIN_NDS_CODE = UI_V1 + "/main-nds-code";
    // ===================================

    private AutoCloseable closeable;
    private ThreadSafeResourceReader reader;
    private MockMvc mockMvc;

    @MockitoBean
    private HttpRequestHelper httpRequestHelper;

    @MockitoBean
    private ObjectMapper mapper;

    @Autowired
    private NdsService service;

    @Autowired
    private ResponseHandler responseHandler;

    @BeforeEach
    void setUp() {
        // если у тебя класс называется иначе — замени
        UiNdsController controller = new NdsUiControllerImpl(service, responseHandler);

        reader = MvcTestUtils.createReader(this);
        closeable = MockitoAnnotations.openMocks(this);

        this.mockMvc = MockMvcBuilders
                .standaloneSetup(controller)
                .defaultResponseCharacterEncoding(StandardCharsets.UTF_8)
                .build();

        // если mapper реально мок — можно убрать, но оставляю как в твоём тесте
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }

    @AfterEach
    void close() throws Exception {
        closeable.close();
    }

    @Test
    @DisplayName("UI test GET {host}" + MAIN_NDS)
    void ui_searchNdsAllTest() throws Exception {
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

        // как в твоём NdsControllerMvcTest: ожидаем 6 элементов
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, MAIN_NDS), 6);
    }

    @Test
    @DisplayName("UI cache test GET {host}" + MAIN_NDS)
    void ui_searchNdsCacheTest() throws Exception {
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

        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, MAIN_NDS + "?rate=0"), 2);
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, MAIN_NDS + "?rate=5"), 1);

        service.cleanCache();

        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, MAIN_NDS + "?rate=5"), 1);

        // “без rate” — как у тебя в тесте, ожидаем полный список
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, MAIN_NDS), 6);
    }

    @Test
    @DisplayName("UI test GET {host}" + MAIN_NDS_CODE)
    void ui_searchNdsCodeCacheTest() throws Exception {
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

        // если у UI-контроллера date обязателен — подставь в URL как в твоём тесте
        String date = "2025-07-21T10:00:03+03:00";

        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, MAIN_NDS_CODE + "?date=" + date), 6);
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, MAIN_NDS_CODE + "?code=N0VA&date=" + date), 1);
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, MAIN_NDS_CODE + "?code=N0VA&rate=0&date=" + date), 1);

        service.cleanCache();

        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, MAIN_NDS_CODE + "?code=N0VA&rate=0&date=" + date), 1);
    }
}
```
