```java

@AutoConfigureMockMvc
@SpringBootTest(classes = {
        SearchRequestProperties.class,
        CacheConfig.class,
        CacheController.class,
        CacheManagerService2.class
})
class CacheControllerMvcTest {

    @Autowired
    private MockMvc mockMvc;

    // ВАЖНО: именно @MockBean, а не @Mock — подменяем бин в контексте Spring
    @MockBean
    private CacheManagerService2 cacheManagerService;

    @Test
    @DisplayName("тест GET {host}/api/v1/cache/status")
    void statusTest() throws Exception {
        // заглушка для сервиса
        when(cacheManagerService.getCacheStatus())
                .thenReturn(Collections.emptyMap());

        mockMvc.perform(get("/api/v1/cache/status")
                        .header(CONTENT_TYPE, APPLICATION_JSON_UTF8_VALUE))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.messages[0].message").value("Статус кэшей"))
                .andExpect(jsonPath("$.messages[0].semantic").value("S"));

        // проверяем, что контроллер действительно дернул сервис
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
                .andExpect(jsonPath("$.messages[0].message")
                        .value("Инвалидация кэшей выполнена."))
                .andExpect(jsonPath("$.messages[0].semantic").value("S"));

        // здесь важно проверить вызов invalidateCache()
        verify(cacheManagerService).invalidateCache();
        verifyNoMoreInteractions(cacheManagerService);
    }
}
```
