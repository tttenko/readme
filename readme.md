```java
 class CommonApplicationConfigTest {

    private final CommonApplicationConfig config = new CommonApplicationConfig();

    @BeforeEach
    void setUp() throws Exception {
        // Эмулируем подстановку @Value("${master-data.search.bufferSize}")
        Field field = CommonApplicationConfig.class.getDeclaredField("bufferSize");
        field.setAccessible(true);
        field.setInt(null, 2); // как будто master-data.search.bufferSize=2
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
        ConnectionProperties props = new ConnectionProperties();
        // значение внутри props теперь используется только как флаг "задано/не задано"
        props.setBufferSize("any");

        int bufferSize = CommonApplicationConfig.getBufferSize(props);

        // ожидаем INITIAL_BUFFER_SIZE * master-data.search.bufferSize (2)
        assertEquals(CommonApplicationConfig.INITIAL_BUFFER_SIZE * 2, bufferSize);
    }

    @Test
    void getBufferSizeWhenNotConfigured() {
        ConnectionProperties props = new ConnectionProperties();
        // bufferSize в props пустой -> возвращаем дефолт

        int bufferSize = CommonApplicationConfig.getBufferSize(props);

        assertEquals(CommonApplicationConfig.INITIAL_BUFFER_SIZE, bufferSize);
    }
}
```
