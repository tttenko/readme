```java
class ApplicationConfigLocalTest {

    /**
     * Хелпер для установки приватного поля через рефлексию
     */
    private void setField(ApplicationConfigLocal config, String name, String value) throws Exception {
        Field field = ApplicationConfigLocal.class.getDeclaredField(name);
        field.setAccessible(true);
        field.set(config, value);
    }

    @Test
    void webClient_withCorrectKeystore() throws Exception {
        // настраиваем конфиг так, как будто @Value всё подставил
        ApplicationConfigLocal applicationConfig = new ApplicationConfigLocal();
        setField(applicationConfig, "filePathKeyStore", "empty-test-keystore.jks");
        setField(applicationConfig, "keyStorePassword", "testpass");
        setField(applicationConfig, "keyStoreType", "JKS");
        // алгоритм можно не задавать, тогда возьмётся дефолтный,
        // но если хочешь — раскомментируй:
        // setField(applicationConfig, "keyStoreAlgorithm", "SunX509");

        SearchRequestProperties searchProps = new SearchRequestProperties();
        searchProps.setBufferSize("1000000");

        WebClient webClient = applicationConfig.webClient(searchProps);

        assertNotNull(webClient);
    }

    @Test
    void webClient_withoutKeystore_throwsException() {
        // поля не трогаем → filePathKeyStore останется null
        ApplicationConfigLocal applicationConfig = new ApplicationConfigLocal();

        SearchRequestProperties searchProps = new SearchRequestProperties();
        searchProps.setBufferSize("1000000");

        assertThrows(IllegalStateException.class,
                () -> applicationConfig.webClient(searchProps));
    }
}
```
