```java
class CommonApplicationConfigTest {

    private final CommonApplicationConfig config = new CommonApplicationConfig();

    // хелпер для установки статического поля bufferSize (эмулируем @Value)
    private void setBufferSize(String value) throws Exception {
        Field field = CommonApplicationConfig.class.getDeclaredField("bufferSize");
        field.setAccessible(true);
        field.set(null, value); // static -> объект = null
    }

    @BeforeEach
    void init() throws Exception {
        // по умолчанию — будто настройки master-data.search.bufferSize нет
        setBufferSize(null);
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
    void getBufferSizeWhenConfigured() throws Exception {
        // эмулируем наличие master-data.search.bufferSize в конфиге
        setBufferSize("any");

        SearchRequestProperties properties = new SearchRequestProperties();
        properties.setBufferSize("2");

        int bufferSize = CommonApplicationConfig.getBufferSize(properties);

        // ожидаем INITIAL_BUFFER_SIZE * properties.bufferSize
        assertEquals(CommonApplicationConfig.INITIAL_BUFFER_SIZE * 2, bufferSize);
    }

    @Test
    void getBufferSizeWhenNotConfigured() {
        // bufferSize == null (установлено в @BeforeEach)
        SearchRequestProperties properties = new SearchRequestProperties();
        properties.setBufferSize("2"); // даже если значение есть, оно игнорируется

        int bufferSize = CommonApplicationConfig.getBufferSize(properties);

        // ожидаем дефолтный размер
        assertEquals(CommonApplicationConfig.INITIAL_BUFFER_SIZE, bufferSize);
    }
}
```
