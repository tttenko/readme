```java

@AutoConfigureMockMvc
@SpringBootTest(classes = {
        CountryController.class,
        CountryService.class,
        SearchRequestProperties.class,
        CountryCacheOps.class,
        CountryMapper.class,
        CacheConfig.class,
        BaseMasterDataRequestService.class,
        CacheGetOrLoadService.class,
        BatchCacheSupport.class,
        LoaderCountryByCode.class
})
class CountryControllerMvcTest {

    @MockBean
    private ObjectMapper mapper;

    @Autowired
    private CountryController controller;

    @Autowired
    private LoaderCountryByCode loaderCountryByCode;

    @MockBean
    private HttpRequestHelper httpRequestHelper;

    @MockBean
    private CacheGetOrLoadService cacheGetOrLoadService;

    private MockMvc mockMvc;
    private ThreadSafeResourceReader reader;
    private AutoCloseable closeable;

    @BeforeEach
    void setUp() {
        closeable = MockitoAnnotations.openMocks(this);

        this.mockMvc = MockMvcBuilders
                .standaloneSetup(controller)
                .defaultResponseCharacterEncoding(StandardCharsets.UTF_8)
                .build();

        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        reader = MvcTestUtils.createReader(this);

        // мок кеш-сервиса поверх реального лоадера
        Mockito.lenient().doAnswer(invocation -> {
            String cacheName = invocation.getArgument(0, String.class);
            List<String> keys = invocation.getArgument(1, List.class);

            if (CountryService.COUNTRY_BY_CODE.equals(cacheName)) {
                return loaderCountryByCode.fetchByKeys(keys);
            }
            return List.of();
        }).when(cacheGetOrLoadService).fetchData(anyString(), anyList());
    }

    @AfterEach
    void tearDown() throws Exception {
        closeable.close();
    }

    @Test
    @DisplayName("test GET {host}/api/v1/info/country")
    void searchAllCountriesTest() throws Exception {
        GetItemsSearchResponse response = reader.readResource(
                "mdresponse/country/country-all.json",
                GetItemsSearchResponse.class
        );

        MvcTestUtils.mockPostResponse(
                "v1/items/byAttrValues",
                response,
                GetItemsSearchResponse.class,
                httpRequestHelper
        );

        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc, "/api/v1/info/country"),
                5  // ожидаемое количество стран в фикстуре
        );
    }

    @Test
    @DisplayName("test GET {host}/api/v1/info/country?countryCode=RU")
    void searchByAlpha2Test() throws Exception {
        GetItemsSearchResponse response = reader.readResource(
                "mdresponse/country/country-RU.json",
                GetItemsSearchResponse.class
        );

        MvcTestUtils.mockPostResponse(
                "v1/items/byAttrValues",
                response,
                GetItemsSearchResponse.class,
                httpRequestHelper
        );

        MvcTestUtils.checkResult(
                MvcTestUtils.performGetOk(mockMvc,
                        "/api/v1/info/country", "countryCode", "RU"),
                1
        );
    }

    // + тест на пустой параметр/отсутствие параметра, который должен вернуть 400
}

```
