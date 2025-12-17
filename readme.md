```java
@ExtendWith(MockitoExtension.class)
class RegionServiceTest {

  @Mock
  private RegionCacheOps regionCacheOps;

  @Mock
  private CacheGetOrLoadService cacheGetOrLoadService;

  @InjectMocks
  private RegionService regionService;

  @Test
  void givenCodes_whenSearchRegionsByCode_thenUseCacheGetOrLoadService() {
    // given
    List<String> codes = List.of("05", "06");
    List<RegionDto> loaded = List.of(new RegionDto(), new RegionDto());

    when(cacheGetOrLoadService.fetchData(RegionService.REGION_BY_CODE, codes))
        .thenReturn(loaded);

    // when
    List<RegionDto> result = regionService.searchRegionsByCode(codes);

    // then
    assertEquals(loaded, result);

    verify(cacheGetOrLoadService).fetchData(RegionService.REGION_BY_CODE, codes);
    verifyNoInteractions(regionCacheOps);
  }

  @Test
  void whenGetAllRegions_thenLoadAllFromRegionCacheOps() {
    // given
    List<RegionDto> all = List.of(new RegionDto(), new RegionDto());

    when(regionCacheOps.loadAllRegions()).thenReturn(all);

    // when
    List<RegionDto> result = regionService.getAllRegions();

    // then
    assertEquals(all, result);

    verify(regionCacheOps).loadAllRegions();
    verifyNoInteractions(cacheGetOrLoadService);
  }
}
```
