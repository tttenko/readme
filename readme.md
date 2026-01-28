```java
@ExtendWith(MockitoExtension.class)
class LoaderCalendDateByDateTest {

    @Mock
    private BaseMasterDataRequestService baseMasterDataRequestService;

    @Mock
    private SearchRequestProperties properties;

    @Mock
    private CalendDateMapper prodCalendDateMapper;

    @InjectMocks
    private LoaderCalendDateByDate loader;

    @Test
    void givenLoader_whenCacheName_thenReturnProdCalendDateByDate() {
        assertEquals(CalendDateService.PROD_CALEND_DATE_BY_DATE, loader.cacheName());
    }

    @Test
    void givenLoader_whenElementType_thenReturnCalendDateDtoClass() {
        assertEquals(CalendDateDto.class, loader.elementType());
    }

    @Test
    void givenCalendDateDto_whenExtractKey_thenReturnDateShort() {
        CalendDateDto dto = new CalendDateDto();
        dto.setDateShort("25.05.2026");

        String key = loader.extractKey(dto);

        assertEquals("25.05.2026", key);
    }

    @Test
    void givenEmptyKeys_whenFetchByKeys_thenReturnEmptyAndSkipBackend() {
        // given
        List<String> keys = List.of();

        // when
        List<CalendDateDto> result = loader.fetchByKeys(keys);

        // then
        assertTrue(result.isEmpty());
        verifyNoInteractions(baseMasterDataRequestService, properties, prodCalendDateMapper);
    }

    @Test
    void givenKeys_whenFetchByKeys_thenDelegateToBackendAndMapResult() {
        // given
        List<String> keys = List.of("25.05.2026", "26.05.2026");
        String slug = "RussianPCDate";

        when(properties.getSlugValueForProdCalendDate()).thenReturn(slug);

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestDataWithAttribute(
                slug,
                keys,
                SearchRequestProperties.Context.BOOK
        )).thenReturn(resp);

        List<CalendDateDto> mapped = List.of(
                CalendDateDto.builder().dateShort("25.05.2026").build(),
                CalendDateDto.builder().dateShort("26.05.2026").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() -> BaseMasterDataRequestService
                    .createResultWithAttribute(resp, prodCalendDateMapper))
                   .thenReturn(mapped);

            // when
            List<CalendDateDto> result = loader.fetchByKeys(keys);

            // then
            assertEquals(mapped, result);

            verify(properties).getSlugValueForProdCalendDate();
            verify(baseMasterDataRequestService, times(1))
                    .requestDataWithAttribute(slug, keys, SearchRequestProperties.Context.BOOK);

            statics.verify(() -> BaseMasterDataRequestService
                    .createResultWithAttribute(resp, prodCalendDateMapper), times(1));

            verifyNoMoreInteractions(baseMasterDataRequestService, properties, prodCalendDateMapper);
        }
    }

    @Test
    void givenTwoSequentialCalls_whenFetchByKeys_thenBackendInvokedTwice_noCaching() {
        // given
        List<String> keys = List.of("25.05.2026");
        String slug = "RussianPCDate";

        when(properties.getSlugValueForProdCalendDate()).thenReturn(slug);

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestDataWithAttribute(
                slug,
                keys,
                SearchRequestProperties.Context.BOOK
        )).thenReturn(resp);

        List<CalendDateDto> mapped = List.of(
                CalendDateDto.builder().dateShort("25.05.2026").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            statics.when(() -> BaseMasterDataRequestService
                    .createResultWithAttribute(resp, prodCalendDateMapper))
                   .thenReturn(mapped);

            // when
            List<CalendDateDto> r1 = loader.fetchByKeys(keys);
            List<CalendDateDto> r2 = loader.fetchByKeys(keys);

            // then
            assertEquals(mapped, r1);
            assertEquals(mapped, r2);

            verify(baseMasterDataRequestService, times(2))
                    .requestDataWithAttribute(slug, keys, SearchRequestProperties.Context.BOOK);

            statics.verify(() -> BaseMasterDataRequestService
                    .createResultWithAttribute(resp, prodCalendDateMapper), times(2));
        }
    }
}

@ExtendWith(MockitoExtension.class)
class CalendDateServiceTest {

    @Mock
    private CacheGetOrLoadService cacheGetOrLoadService;

    @InjectMocks
    private CalendDateService calendDateService;

    @Test
    void givenDates_whenSearchByDates_thenUseCacheGetOrLoadService() {
        // given
        List<String> dates = List.of("25.05.2026", "26.05.2026");
        List<CalendDateDto> loaded = List.of(new CalendDateDto(), new CalendDateDto());

        when(cacheGetOrLoadService.fetchData(CalendDateService.PROD_CALEND_DATE_BY_DATE, dates))
            .thenReturn(loaded);

        // when
        List<CalendDateDto> result = calendDateService.searchByDates(dates);

        // then
        assertEquals(loaded, result);
        verify(cacheGetOrLoadService)
            .fetchData(CalendDateService.PROD_CALEND_DATE_BY_DATE, dates);
        verifyNoMoreInteractions(cacheGetOrLoadService);
    }

    @Test
    void givenNullDates_whenSearchByDates_thenNormalizeToMoscowTodayAndUseCache() {
        // given
        String todayMoscow = LocalDate.now(ZoneId.of("Europe/Moscow"))
            .format(DateTimeFormatter.ofPattern("dd.MM.yyyy"));

        List<CalendDateDto> loaded = List.of(new CalendDateDto());

        when(cacheGetOrLoadService.fetchData(
            CalendDateService.PROD_CALEND_DATE_BY_DATE,
            List.of(todayMoscow)
        )).thenReturn(loaded);

        // when
        List<CalendDateDto> result = calendDateService.searchByDates(null);

        // then
        assertEquals(loaded, result);
        verify(cacheGetOrLoadService)
            .fetchData(CalendDateService.PROD_CALEND_DATE_BY_DATE, List.of(todayMoscow));
        verifyNoMoreInteractions(cacheGetOrLoadService);
    }

    @Test
    void givenEmptyDates_whenSearchByDates_thenNormalizeToMoscowTodayAndUseCache() {
        // given
        String todayMoscow = LocalDate.now(ZoneId.of("Europe/Moscow"))
            .format(DateTimeFormatter.ofPattern("dd.MM.yyyy"));

        List<CalendDateDto> loaded = List.of(new CalendDateDto());

        when(cacheGetOrLoadService.fetchData(
            CalendDateService.PROD_CALEND_DATE_BY_DATE,
            List.of(todayMoscow)
        )).thenReturn(loaded);

        // when
        List<CalendDateDto> result = calendDateService.searchByDates(List.of());

        // then
        assertEquals(loaded, result);
        verify(cacheGetOrLoadService)
            .fetchData(CalendDateService.PROD_CALEND_DATE_BY_DATE, List.of(todayMoscow));
        verifyNoMoreInteractions(cacheGetOrLoadService);
    }
}

@AutoConfigureMockMvc
@SpringBootTest(classes = {
    CalendDateControllerImpl.class,
    CalendDateService.class,
    SearchRequestProperties.class,
    CalendDateMapper.class,
    CacheConfig.class,
    BaseMasterDataRequestService.class,
    CacheGetOrLoadService.class,
    BatchCacheSupport.class,
    LoaderCalendDateByDate.class,
    ResponseHandler.class
})
class CalendDateControllerMvcTest {

    @MockitoBean
    private ObjectMapper mapper;

    @Autowired
    private CalendDateController controller;

    @Autowired
    private LoaderCalendDateByDate loaderCalendDateByDate;

    @MockitoBean
    private HttpRequestHelper httpRequestHelper;

    @MockitoBean
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

        // Прокидываем fetchData -> loader.fetchByKeys (как в CountryControllerMvcTest)
        Mockito.lenient().doAnswer(invocation -> {
            String cacheName = invocation.getArgument(0, String.class);
            List<String> keys = invocation.getArgument(1, List.class);

            if (CalendDateService.PROD_CALEND_DATE_BY_DATE.equals(cacheName)) {
                return loaderCalendDateByDate.fetchByKeys(keys);
            }
            return List.of();
        }).when(cacheGetOrLoadService).fetchData(anyString(), anyList());
    }

    @AfterEach
    void tearDown() throws Exception {
        closeable.close();
    }

    @Test
    @DisplayName("test GET {host}/api/v1/info/prod_calend_date/{date}")
    void searchByDatePathVariableTest() throws Exception {
        GetItemsSearchResponse response = reader.readResource(
            "mdresponse/calend_date/prod_calend_date-25.05.2026.json",
            GetItemsSearchResponse.class
        );

        MvcTestUtils.mockPostResponse(
            "v1/items/byAttrValues",
            response,
            GetItemsSearchResponse.class,
            httpRequestHelper
        );

        MvcTestUtils.checkResult(
            MvcTestUtils.performGetOk(mockMvc, "/api/v1/info/prod_calend_date/25.05.2026"),
            1
        );
    }

    @Test
    @DisplayName("GET /api/v1/info/prod_calend_date/{date} invalid pattern -> 400")
    void searchByDateInvalidPatternReturnsBadRequest() throws Exception {
        var result = MvcTestUtils.performGet(mockMvc, "/api/v1/info/prod_calend_date/25-05-2026")
            .andReturn();

        assertEquals(400, result.getResponse().getStatus());

        // обычно при @Validated + @Pattern на PathVariable будет ConstraintViolationException
        assertEquals(ConstraintViolationException.class, result.getResolvedException().getClass());
    }
}

{
  "messages": [
    { "message": "Поиск выполнен", "description": "Найдено записей: 1", "semantic": "S" }
  ],
  "count": 1,
  "data": [
    {
      "items": [
        {
          "item": {
            "id": "a1b2c3d4-e5f6-7890-1111-222233334444",
            "slug": "25.05.2026",
            "name": "Нерабочий праздничный день",
            "description": "4"
          },
          "values": [
            {
              "attribute": {
                "id": "70c4d635-7e74-4c70-a6c0-d779b7b93b08",
                "slug": "date",
                "type": "DATE",
                "value": "2026-05-25 00:00:00.0"
              },
              "valueRef": null
            },
            {
              "attribute": {
                "id": "3f151a3b-5cb1-4ca2-b65a-03b359f0f18d",
                "slug": "dateType",
                "type": "FROM_REFERENCE",
                "value": "4 - Нерабочий праздничный день"
              },
              "valueRef": {
                "valueRefItem": {
                  "item": {
                    "slug": "4",
                    "name": "Нерабочий праздничный день, согласно части первой ст. 112 ТК РФ"
                  }
                }
              }
            }
          ]
        }
      ]
    }
  ]
}
```
