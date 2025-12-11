```java
class ApplicationConfigLocalTest {

    @Test
    void objectMapper() {
        // ObjectMapper теперь создаётся в CommonApplicationConfig без зависимостей
        CommonApplicationConfig commonConfig = new CommonApplicationConfig();

        ObjectMapper objectMapper = commonConfig.objectMapper();

        assertNotNull(objectMapper);
    }

    @Test
    void webClient_withCorrectKeystore() {
        // настройки подключения (keystore ок)
        ConnectionProperties props = new ConnectionProperties();
        props.setFilePathKeyStore("empty-test-keystore.jks");
        props.setFilePathTrustStore("empty-test-keystore.jks");
        props.setKeyStorePassword("testpass");
        props.setKeyStoreType("JKS");

        // настройки поиска (bufferSize для WebClient)
        SearchRequestProperties searchProps = new SearchRequestProperties();
        searchProps.setBufferSize("1000000");

        ApplicationConfigLocal applicationConfig = new ApplicationConfigLocal();

        WebClient webClient = applicationConfig.webClient(props, searchProps);

        assertNotNull(webClient);
    }

    @Test
    void webClient_withIncorrectKeystore() {
        // настройки подключения (keystore с неверным путём,
        // но метод должен вернуть WebClient даже при исключении внутри try)
        ConnectionProperties props = new ConnectionProperties();
        props.setFilePathKeyStore("empty-test-keystore1.jks");
        props.setFilePathTrustStore("empty-test-keystore1.jks");
        props.setKeyStorePassword("testpass");
        props.setKeyStoreType("JKS");

        // настройки поиска
        SearchRequestProperties searchProps = new SearchRequestProperties();
        searchProps.setBufferSize("1000000");

        ApplicationConfigLocal applicationConfig = new ApplicationConfigLocal();

        WebClient webClient = applicationConfig.webClient(props, searchProps);

        assertNotNull(webClient);
    }
}
```
