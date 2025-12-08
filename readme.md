```java

@SpringBootTest
@EnableAutoConfiguration
@ContextConfiguration(classes = ConnectionProperties.class)
@TestPropertySource(properties = {
        "master-data.connection.keyStoreType=PKCS12",
        "master-data.connection.keyStoreAlgorithm=SunX509",
        "master-data.connection.filePathKeyStore=cert.pfx",
        "master-data.connection.filePathTrustStore=cert.pfx",
        "master-data.connection.keyStorePassword=q1w2e3r4",
        "master-data.connection.bufferSize=40000"
})
class ConnectionPropertiesTest {

    @Autowired
    private ConnectionProperties connectionProperties;

    @Test
    void shouldLoadPropertiesFromYml() {
        assertNotNull(connectionProperties.getFilePathKeyStore());
        assertNotNull(connectionProperties.getFilePathTrustStore());
        assertNotNull(connectionProperties.getKeyStoreType());
        assertNotNull(connectionProperties.getKeyStorePassword());
        assertNotNull(connectionProperties.getKeyStoreAlgorithm());
    }
}
```
