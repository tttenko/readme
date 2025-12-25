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

    private MockMvc mockMvc;

    @MockitoBean
    private HttpRequestHelper httpRequestHelper;

    private ThreadSafeResourceReader reader;

    @BeforeEach
    void setUp() {
        var controller = new NdsUiControllerImpl(service, responseHandler);

        reader = MvcTestUtils.createReader(this);
        closeable = MockitoAnnotations.openMocks(this);

        this.mockMvc = MockMvcBuilders
                .standaloneSetup(controller)
                .defaultResponseCharacterEncoding(StandardCharsets.UTF_8)
                .build();

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
    @DisplayName("test GET {host}/ui/v1/main-nds (cache + фильтры)")
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

        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds?rate=0"), 2);
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds?rate=5"), 1);

        service.cleanCache();

        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds?rate=5"), 1);
        MvcTestUtils.checkResult(MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds"), 6);

        // если вдруг начнёт падать парсинг из-за "+03:00", замени на "%2B03:00"
        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds?rate=5&date=2025-07-21T10:00:00+03:00"),
                1
        );
    }

    @Test
    @DisplayName("test GET {host}/ui/v1/main-nds-code")
    void searchUiNdsCodeTest() throws Exception {
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
                MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds-code?date=" + date),
                6
        );

        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds-code?code=NOVAT&date=" + date),
                1
        );

        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds-code?code=NOVAT&rate=0&date=" + date),
                1
        );

        service.cleanCache();

        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc, "/ui/v1/main-nds-code?code=NOVAT&rate=0&date=" + date),
                1
        );
    }
}
```
