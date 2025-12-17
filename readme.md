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

  private MockMvc mockMvc;
  private ThreadSafeResourceReader reader;
  private AutoCloseable closeable;

  @BeforeEach
  void setUp() {
    closeable = org.mockito.MockitoAnnotations.openMocks(this);

    this.mockMvc = standaloneSetup(controller)
        .defaultResponseCharacterEncoding(StandardCharsets.UTF_8)
        .build();

    mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    reader = MvcTestUtils.createReader(this);

    org.mockito.Mockito.lenient().doAnswer(invocation -> {
      String cacheName = invocation.getArgument(0, String.class);
      List<String> keys = invocation.getArgument(1, List.class);

      if (RegionService.REGION_BY_CODE.equals(cacheName)) {
        return loaderRegionByCode.fetchByKeys(keys);
      }
      return List.of();
    }).when(cacheGetOrLoadService).fetchData(org.mockito.Mockito.anyString(), org.mockito.Mockito.anyList());
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
  void searchByRegionCodeTest() throws Exception {
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
    assertEquals(
        MissingServletRequestParameterException.class,
        result.getResolvedException().getClass()
    );
  }
}
```
