```java



@ExtendWith(MockitoExtension.class)
class RegionCacheOpsTest {

  @Mock
  private BaseMasterDataRequestService baseMasterDataRequestService;

  @Mock
  private SearchRequestProperties properties;

  @Mock
  private RegionMapper regionMapper;

  @InjectMocks
  private RegionCacheOps regionCacheOps;

  @Test
  void givenBackendResponse_whenLoadAllRegions_thenReturnMappedList() {
    // given
    String slug = "region";

    when(properties.getSlugValueForRegion()).thenReturn(slug);

    GetItemsSearchResponse resp = mock(GetItemsSearchResponse.class);
    when(baseMasterDataRequestService.requestData(slug, null, SearchRequestProperties.Context.BOOK))
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
      List<RegionDto> result = regionCacheOps.loadAllRegions();

      // then
      assertEquals(mapped, result);

      verify(properties).getSlugValueForRegion();
      verify(baseMasterDataRequestService).requestData(slug, null, SearchRequestProperties.Context.BOOK);

      statics.verify(() -> BaseMasterDataRequestService.createResult(Mockito.eq(resp), Mockito.any()));

      verifyNoMoreInteractions(baseMasterDataRequestService, properties, regionMapper);
    }
  }

  @Test
  void givenEmptyBackendResponse_whenLoadAllRegions_thenReturnEmptyList() {
    // given
    String slug = "region";

    when(properties.getSlugValueForRegion()).thenReturn(slug);

    GetItemsSearchResponse resp = mock(GetItemsSearchResponse.class);
    when(baseMasterDataRequestService.requestData(slug, null, SearchRequestProperties.Context.BOOK))
        .thenReturn(resp);

    List<RegionDto> mapped = List.of(); // пустой список

    try (MockedStatic<BaseMasterDataRequestService> statics =
             Mockito.mockStatic(BaseMasterDataRequestService.class)) {

      statics.when(() -> BaseMasterDataRequestService.createResult(Mockito.eq(resp), Mockito.any()))
          .thenReturn(mapped);

      // when
      List<RegionDto> result = regionCacheOps.loadAllRegions();

      // then
      assertEquals(mapped, result);

      verify(properties).getSlugValueForRegion();
      verify(baseMasterDataRequestService).requestData(slug, null, SearchRequestProperties.Context.BOOK);

      statics.verify(() -> BaseMasterDataRequestService.createResult(Mockito.eq(resp), Mockito.any()));

      verifyNoMoreInteractions(baseMasterDataRequestService, properties, regionMapper);
    }
  }
}
```
