```java
class CommonApplicationConfigTest {

    private CommonApplicationConfig config;
    private SearchRequestProperties searchProps;

    @BeforeEach
    void setUp() {
        searchProps = new SearchRequestProperties();
        // если у класса другой пакет/конструктор — подправь импорт/вызов
        config = new CommonApplicationConfig(searchProps);
    }

    @Test
    void objectMapperBeanCreated() {
        ObjectMapper mapper = config.objectMapper();
        assertNotNull(mapper);
    }

    @Test
    void prepareRequestBeanCreated() {
        ObjectMapper mapper = new ObjectMapper();
        WebClient webClient = WebClient.create("http://localhost");

        HttpRequestHelper helper = config.prepareRequest(mapper, webClient);

        assertNotNull(helper);
    }

    @Test
    void getBufferSizeWhenConfigured() {
        // в properties пришло значение "2"
        searchProps.setBufferSize("2");

        int bufferSize = config.getBufferSize();

        assertEquals(CommonApplicationConfig.INITIAL_BUFFER_SIZE * 2, bufferSize);
    }

    @Test
    void getBufferSizeWhenNotConfigured() {
        // значение не задано (null) — должен вернуться дефолт
        searchProps.setBufferSize(null);

        int bufferSize = config.getBufferSize();

        assertEquals(CommonApplicationConfig.INITIAL_BUFFER_SIZE, bufferSize);
    }
}
```
