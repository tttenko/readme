```java
 class CommonApplicationConfigTest {

    private final CommonApplicationConfig config = new CommonApplicationConfig();

    // Хелпер: устанавливаем значение статического поля bufferSize через рефлексию
    private void setBufferSize(String value) throws Exception {
        Field field = CommonApplicationConfig.class.getDeclaredField("bufferSize");
        field.setAccessible(true);
        field.set(null, value); // т.к. поле static
    }

    @BeforeEach
    void init() throws Exception {
        // по умолчанию очищаем bufferSize (как будто настройки нет)
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
        // эмулируем master-data.search.bufferSize=2
        setBufferSize("2");

        int bufferSize = CommonApplicationConfig.getBufferSize();

        assertEquals(CommonApplicationConfig.INITIAL_BUFFER_SIZE * 2, bufferSize);
    }

    @Test
    void getBufferSizeWhenNotConfigured() {
        // bufferSize = null (из @BeforeEach)

        int bufferSize = CommonApplicationConfig.getBufferSize();

        assertEquals(CommonApplicationConfig.INITIAL_BUFFER_SIZE, bufferSize);
    }
}
```
