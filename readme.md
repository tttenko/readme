```java



@ExtendWith(MockitoExtension.class)
class LoaderRegionByCodeTest {

  @Mock
  private BaseMasterDataRequestService baseMasterDataRequestService;

  @Mock
  private SearchRequestProperties properties;

  @Mock
  private RegionMapper regionMapper;

  @InjectMocks
  private LoaderRegionByCode loader;

  @Test
  void givenLoader_whenCacheName_thenReturnRegionByCode() {
    assertEquals(RegionService.REGION_BY_CODE, loader.cacheName());
  }

  @Test
  void givenLoader_whenElementType_thenReturnRegionDtoClass() {
    assertEquals(RegionDto.class, loader.elementType());
  }

  @Test
  void givenRegionDto_whenExtractKey_thenReturnRegionCode() {
    RegionDto dto = new RegionDto();
    dto.setRegionCode("05");

    String key = loader.extractKey(dto);

    assertEquals("05", key);
  }

  @Test
  void givenEmptyKeys_whenFetchByKeys_thenReturnEmptyAndSkipBackend() {
    // given
    List<String> keys = List.of();

    // when
    List<RegionDto> result = loader.fetchByKeys(keys);

    // then
    assertTrue(result.isEmpty());
    verifyNoInteractions(baseMasterDataRequestService, properties, regionMapper);
  }

  @Test
  void givenCodes_whenFetchByKeys_thenDelegateToBackendAndMapResult() {
    // given
    List<String> codes = List.of("05", "06");
    String slug = "region";

    when(properties.getSlugValueForRegion()).thenReturn(slug);

    GetItemsSearchResponse resp = new GetItemsSearchResponse();
    when(baseMasterDataRequestService.requestData(slug, codes, SearchRequestProperties.Context.BOOK))
        .thenReturn(resp);

    List<RegionDto> mapped = List.of(
        RegionDto.builder().regionCode("05").build(),
        RegionDto.builder().regionCode("06").build()
    );

    try (MockedStatic<BaseMasterDataRequestService> statics =
             Mockito.mockStatic(BaseMasterDataRequestService.class)) {

      statics.when(() -> BaseMasterDataRequestService.createResult(Mockito.eq(resp), Mockito.any()))
          .thenReturn(mapped);

      // when
      List<RegionDto> result = loader.fetchByKeys(codes);

      // then
      assertEquals(mapped, result);

      verify(properties).getSlugValueForRegion();
      verify(baseMasterDataRequestService, times(1))
          .requestData(slug, codes, SearchRequestProperties.Context.BOOK);

      statics.verify(() -> BaseMasterDataRequestService.createResult(Mockito.eq(resp), Mockito.any()),
          times(1));

      verifyNoMoreInteractions(baseMasterDataRequestService, properties, regionMapper);
    }
  }

  @Test
  void givenTwoSequentialCalls_whenFetchByKeys_thenBackendInvokedTwice_noCaching() {
    // given
    List<String> codes = List.of("01");
    String slug = "region";

    when(properties.getSlugValueForRegion()).thenReturn(slug);

    GetItemsSearchResponse resp = new GetItemsSearchResponse();
    when(baseMasterDataRequestService.requestData(slug, codes, SearchRequestProperties.Context.BOOK))
        .thenReturn(resp);

    List<RegionDto> mapped = List.of(
        RegionDto.builder().regionCode("01").build()
    );

    try (MockedStatic<BaseMasterDataRequestService> statics =
             Mockito.mockStatic(BaseMasterDataRequestService.class)) {

      statics.when(() -> BaseMasterDataRequestService.createResult(Mockito.eq(resp), Mockito.any()))
          .thenReturn(mapped);

      // when
      List<RegionDto> r1 = loader.fetchByKeys(codes);
      List<RegionDto> r2 = loader.fetchByKeys(codes);

      // then
      assertEquals(mapped, r1);
      assertEquals(mapped, r2);

      verify(properties, times(2)).getSlugValueForRegion();
      verify(baseMasterDataRequestService, times(2))
          .requestData(slug, codes, SearchRequestProperties.Context.BOOK);

      statics.verify(() -> BaseMasterDataRequestService.createResult(Mockito.eq(resp), Mockito.any()),
          times(2));
    }
  }
}
```
