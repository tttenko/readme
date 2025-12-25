```java
p@SpringBootTest(classes = {
        // набор оставь такой же, как у тебя в текущих тестах
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
class NdsBothControllersMvcTest {

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

    private ThreadSafeResourceReader reader;

    enum Variant {
        API("/api/v1") {
            @Override Object controller(NdsService s, ResponseHandler rh) {
                return new NdsControllerImpl(s, rh);
            }
        },
        UI("/ui/v1") {
            @Override Object controller(NdsService s, ResponseHandler rh) {
                return new NdsUiControllerImpl(s, rh);
            }
        };

        final String basePath;
        Variant(String basePath) { this.basePath = basePath; }
        abstract Object controller(NdsService s, ResponseHandler rh);
    }

    @BeforeEach
    void setUp() {
        reader = MvcTestUtils.createReader(this);
        closeable = MockitoAnnotations.openMocks(this);

        // чтобы мок пост-запроса совпадал с тем, куда реально ходит сервис
        when(searchRequestProperties.getGetListByAttrValuesUri(SearchRequestProperties.Context.BOOK))
                .thenReturn("v1/items/byAttrValues");

        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

        // чтобы тесты не влияли друг на друга кэшем
        service.cleanCache();
    }

    @AfterEach
    void tearDown() throws Exception {
        closeable.close();
    }

    private MockMvc mvcFor(Variant v) {
        return MockMvcBuilders
                .standaloneSetup(v.controller(service, responseHandler))
                .defaultResponseCharacterEncoding(StandardCharsets.UTF_8)
                .build();
    }

    private MvcResult performGetOk(MockMvc mvc, String path) throws Exception {
        return mvc.perform(get(path))
                .andExpect(status().isOk())
                .andReturn();
    }

    private MvcResult performGetOk(MockMvc mvc, String path, String... params) throws Exception {
        var req = get(path);
        // params: "k1","v1","k2","v2" ...
        for (int i = 0; i < params.length; i += 2) {
            req = req.param(params[i], params[i + 1]);
        }
        return mvc.perform(req)
                .andExpect(status().isOk())
                .andReturn();
    }

    @ParameterizedTest
    @EnumSource(Variant.class)
    @DisplayName("GET main-nds: all")
    void searchNdsAll(Variant v) throws Exception {
        MockMvc mockMvc = mvcFor(v);

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
                performGetOk(mockMvc, v.basePath + "/main-nds"),
                6
        );
    }

    @ParameterizedTest
    @EnumSource(Variant.class)
    @DisplayName("GET main-nds: cache + params")
    void searchNdsCache(Variant v) throws Exception {
        MockMvc mockMvc = mvcFor(v);

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
        MvcTestUtils.checkResult(performGetOk(mockMvc, v.basePath + "/main-nds", "rate", "0"), 2);
        MvcTestUtils.checkResult(performGetOk(mockMvc, v.basePath + "/main-nds", "rate", "5"), 1);

        // чистим кэш
        service.cleanCache();

        // повторный вызов после чистки
        MvcTestUtils.checkResult(performGetOk(mockMvc, v.basePath + "/main-nds", "rate", "5"), 1);
        MvcTestUtils.checkResult(performGetOk(mockMvc, v.basePath + "/main-nds"), 6);

        // date передаём через param (без ручного %2B и без 400)
        MvcTestUtils.checkResult(
                performGetOk(
                        mockMvc,
                        v.basePath + "/main-nds",
                        "rate", "5",
                        "date", "2025-07-21T10:00:00+03:00"
                ),
                1
        );
    }

    @ParameterizedTest
    @EnumSource(Variant.class)
    @DisplayName("GET main-nds-code: cache + params")
    void searchNdsCodeCache(Variant v) throws Exception {
        MockMvc mockMvc = mvcFor(v);

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

        String date = "2025-07-21T10:00:00+03:00";

        MvcTestUtils.checkResult(
                performGetOk(mockMvc, v.basePath + "/main-nds-code", "date", date),
                6
        );

        MvcTestUtils.checkResult(
                performGetOk(mockMvc, v.basePath + "/main-nds-code", "code", "NOVAT", "date", date),
                1
        );

        MvcTestUtils.checkResult(
                performGetOk(mockMvc, v.basePath + "/main-nds-code", "code", "NOVAT", "rate", "0", "date", date),
                1
        );

        service.cleanCache();

        MvcTestUtils.checkResult(
                performGetOk(mockMvc, v.basePath + "/main-nds-code", "code", "NOVAT", "rate", "0", "date", date),
                1
        );
    }
}
```
