```java
 class CommonApplicationConfigTest {

    private final CommonApplicationConfig config = new CommonApplicationConfig();

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
        props.setBufferSize("2");

        int bufferSize = CommonApplicationConfig.getBufferSize(props);

        assertEquals(CommonApplicationConfig.INITIAL_BUFFER_SIZE * 2, bufferSize);
    }

    @Test
    void getBufferSizeWhenNotConfigured() {
        ConnectionProperties props = new ConnectionProperties();

        int bufferSize = CommonApplicationConfig.getBufferSize(props);

        assertEquals(CommonApplicationConfig.INITIAL_BUFFER_SIZE, bufferSize);
    }
}

class ApplicationConfigTest {

    private final ApplicationConfig config = new ApplicationConfig();

    @Test
    void webClientBeanCreated() {
        ConnectionProperties props = new ConnectionProperties();
        props.setBufferSize("2"); // чтобы прошёл расчёт буфера

        WebClient webClient = config.webClient(props);

        assertNotNull(webClient);
    }
}
```
