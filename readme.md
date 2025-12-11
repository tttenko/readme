```java
class ApplicationConfigLocalTest {

    @Test
    void webClient_withCorrectKeystore() {
        ConnectionProperties props = new ConnectionProperties();
        props.setFilePathKeyStore("empty-test-keystore.jks");
        props.setFilePathTrustStore("empty-test-keystore.jks");
        props.setKeyStorePassword("testpass");
        props.setKeyStoreType("JKS");

        SearchRequestProperties searchProps = new SearchRequestProperties();
        searchProps.setBufferSize("1000000");

        ApplicationConfigLocal applicationConfig = new ApplicationConfigLocal();

        WebClient webClient = applicationConfig.webClient(props, searchProps);

        assertNotNull(webClient);
    }

    @Test
    void webClient_withoutKeystore_throwsException() {
        ConnectionProperties props = new ConnectionProperties();
        // специально ничего не задаём / или задаём пустые строки

        SearchRequestProperties searchProps = new SearchRequestProperties();
        searchProps.setBufferSize("1000000");

        ApplicationConfigLocal applicationConfig = new ApplicationConfigLocal();

        assertThrows(IllegalStateException.class,
                () -> applicationConfig.webClient(props, searchProps));
    }
}
```
