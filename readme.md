```java
class ApplicationConfigLocalTest {

    @Test
    void objectMapper() {
        // теперь CommonApplicationConfig зависит от SearchRequestProperties
        SearchRequestProperties searchProps = new SearchRequestProperties();
        // при желании можешь задать searchProps.setBufferSize("2");
        CommonApplicationConfig commonConfig = new CommonApplicationConfig(searchProps);

        ObjectMapper objectMapper = commonConfig.objectMapper();

        assertNotNull(objectMapper);
    }

    @Test
    void webClient_withCorrectKeystore() {
        ConnectionProperties props = new ConnectionProperties();
        props.setFilePathKeyStore("empty-test-keystore.jks");
        props.setFilePathTrustStore("empty-test-keystore.jks");
        props.setKeyStorePassword("testpass");
        props.setKeyStoreType("JKS");
        props.setBufferSize("1000000");

        ApplicationConfigLocal applicationConfig = new ApplicationConfigLocal();
        WebClient webClient = applicationConfig.webClient(props);

        assertNotNull(webClient);
    }

    @Test
    void webClient_withIncorrectKeystore() {
        ConnectionProperties props = new ConnectionProperties();
        props.setFilePathKeyStore("empty-test-keystore1.jks");
        props.setFilePathTrustStore("empty-test-keystore1.jks");
        props.setKeyStorePassword("testpass");
        props.setKeyStoreType("JKS");
        props.setBufferSize("1000000");

        ApplicationConfigLocal applicationConfig = new ApplicationConfigLocal();
        WebClient webClient = applicationConfig.webClient(props);

        assertNotNull(webClient);
    }
}
```
