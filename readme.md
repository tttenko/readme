```java

@WebMvcTest(CacheController.class)
class CacheControllerMvcTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private CacheManagerService cacheManagerService;

    @Test
    @DisplayName("тест GET {host}/api/v1/cache/status")
    void statusTest() throws Exception {
        // given: подготавливаем DTO, которое вернёт сервис
        CacheStatusDto cacheDto = CacheStatusDto.builder()
                .name("tb_req_all")
                .type("CaffeineCache")
                .estimatedSize(0L)
                .hitCount(0L)
                .missCount(0L)
                .hitRate(0.0)
                .missRate(0.0)
                .loadSuccess(0L)
                .loadFailure(0L)
                .evictionCount(0L)
                .build();

        CacheStatusResponse response = CacheStatusResponse.builder()
                .lastManualInvalidation("test-time")
                .ttl(123L)
                .caches(List.of(cacheDto))
                .build();

        when(cacheManagerService.getCacheStatus()).thenReturn(response);

        // when / then
        mockMvc.perform(get("/api/v1/cache/status")
                        .header(CONTENT_TYPE, APPLICATION_JSON_UTF8_VALUE))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.messages[0].message").value("Cache status"))
                .andExpect(jsonPath("$.messages[0].semantic").value("S"))
                // проверяем, что DTO корректно улетело в data
                .andExpect(jsonPath("$.data.lastManualInvalidation").value("test-time"))
                .andExpect(jsonPath("$.data.ttl").value(123))
                .andExpect(jsonPath("$.data.caches").isArray())
                .andExpect(jsonPath("$.data.caches[0].name").value("tb_req_all"))
                .andExpect(jsonPath("$.data.caches[0].type").value("CaffeineCache"));

        verify(cacheManagerService).getCacheStatus();
        verifyNoMoreInteractions(cacheManagerService);
    }

    @Test
    @DisplayName("тест GET {host}/api/v1/cache/invalidate")
    void invalidateTest() throws Exception {
        mockMvc.perform(get("/api/v1/cache/invalidate")
                        .header(CONTENT_TYPE, APPLICATION_JSON_UTF8_VALUE))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.messages[0].message").value("Cache invalidation completed."))
                .andExpect(jsonPath("$.messages[0].semantic").value("S"));

        verify(cacheManagerService).invalidateCache();
        verifyNoMoreInteractions(cacheManagerService);
    }
}

```
