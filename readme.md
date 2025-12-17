```java
@SpringBootTest(classes = {
    RegionController.class,
    RegionService.class,
    SearchRequestProperties.class,
    RegionCacheOps.class,
    RegionMapper.class,
    CacheConfig.class,
    BaseMasterDataRequestService.class,
    CacheGetOrLoadService.class,
    BatchCacheSupport.class,
    LoaderRegionByCode.class,
    ResponseHandler.class
})
class RegionControllerMvcTest {

  @MockitoBean
  private ObjectMapper mapper;

  @Autowired
  private RegionController controller;

  @Autowired
  private LoaderRegionByCode loaderRegionByCode;

  @MockitoBean
  private HttpRequestHelper httpRequestHelper;

  @MockitoBean
  private CacheGetOrLoadService cacheGetOrLoadService;

  @MockitoBean
  private SearchRequestProperties properties;

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

    // Важно: чтобы reader нормально десериализовывал фикстуры
    mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    reader = MvcTestUtils.createReader(this);

    // Критично: чтобы URL, который реально дергает BaseMasterDataRequestService, совпал со стабом mockPostResponse
    Mockito.when(properties.getGetListByAttrValuesUri(SearchRequestProperties.Context.BOOK))
        .thenReturn("v1/items/byAttrValues");

    // Критично: чтобы dictionaryName/slug был не null
    Mockito.when(properties.getSlugValueForRegion()).thenReturn("region");

    // Стаб как у стран: для query-поиска дергаем loader
    lenient().doAnswer(invocation -> {
      String cacheName = invocation.getArgument(0, String.class);
      List<String> keys = invocation.getArgument(1, List.class);

      if (RegionService.REGION_BY_CODE.equals(cacheName)) {
        return loaderRegionByCode.fetchByKeys(keys);
      }
      return List.of();
    }).when(cacheGetOrLoadService).fetchData(anyString(), anyList());
  }

  @AfterEach
  void tearDown() throws Exception {
    closeable.close();
  }

  @Test
  @DisplayName("test GET {host}/api/v1/info/region_code/all")
  void searchAllRegionsTest() throws Exception {
    GetItemsSearchResponse response = reader.readResource(
        "mdresponse/region/region-all.json",
        GetItemsSearchResponse.class
    );

    // Вариант 1: как у стран (если mockPostResponse матчится по URL)
    MvcTestUtils.mockPostResponse(
        "v1/items/byAttrValues",
        response,
        GetItemsSearchResponse.class,
        httpRequestHelper
    );

    MvcTestUtils.checkResult(
        MvcTestUtils.performGetOk(mockMvc, "/api/v1/info/region_code/all"),
        2
    );
  }

  @Test
  @DisplayName("test GET {host}/api/v1/info/region_code?regionCode=05")
  void searchByCodeTest() throws Exception {
    GetItemsSearchResponse response = reader.readResource(
        "mdresponse/region/region-05.json",
        GetItemsSearchResponse.class
    );

    MvcTestUtils.mockPostResponse(
        "v1/items/byAttrValues",
        response,
        GetItemsSearchResponse.class,
        httpRequestHelper
    );

    MvcTestUtils.checkResult(
        MvcTestUtils.performGetOk(mockMvc, "/api/v1/info/region_code", "regionCode", "05"),
        1
    );
  }

  @Test
  @DisplayName("Get /api/v1/info/region_code без параметров")
  void searchRegionsWithoutCodesReturnsBadRequest() throws Exception {
    MvcResult result = MvcTestUtils.performGet(mockMvc, "/api/v1/info/region_code").andReturn();

    assertEquals(400, result.getResponse().getStatus());
    assertEquals(MissingServletRequestParameterException.class, result.getResolvedException().getClass());
  }
}
```
