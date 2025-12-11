```java
@Slf4j
@Configuration
@Profile("local")
public class ApplicationConfigLocal {

    /**
     * Создаёт и настраивает {@link WebClient} для взаимодействия с сервисом
     * мастер-данных в локальной среде с поддержкой SSL.
     *
     * @param props      свойства подключения (keystore)
     * @param properties свойства запросов (размер буфера и т.п.)
     * @return настроенный экземпляр WebClient
     */
    @Bean
    public WebClient webClient(ConnectionProperties props, SearchRequestProperties properties) {
        // SSL обязательно для local → если не задан путь, падаем сразу
        if (isEmpty(props.getFilePathTrustStore()) || isEmpty(props.getFilePathKeyStore())) {
            throw new IllegalStateException(
                    "SSL is required for 'local' profile: master-data.connection.filePathKeyStore/filePathTrustStore must be set");
        }

        ClassPathResource resource = new ClassPathResource(props.getFilePathKeyStore());
        HttpClient httpClient;

        try {
            InputStream pfxStream = resource.getInputStream(); // Получаем поток ресурса

            KeyStore keyStore = KeyStore.getInstance(
                    defaultIfNull(props.getKeyStoreType(), KeyStore.getDefaultType())
            );
            keyStore.load(pfxStream, props.getKeyStorePassword().toCharArray()); // Загружаем данные из потока

            KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(
                    defaultIfNull(props.getKeyStoreAlgorithm(), KeyManagerFactory.getDefaultAlgorithm())
            );
            keyManagerFactory.init(keyStore, props.getKeyStorePassword().toCharArray());

            SslContext sslContext = SslContextBuilder.forClient()
                    .keyManager(keyManagerFactory)
                    .trustManager(InsecureTrustManagerFactory.INSTANCE)
                    .build();

            httpClient = HttpClient.create()
                    .secure(t -> t.sslContext(sslContext));
        } catch (Exception e) {
            log.error("Failed to configure SSL for local WebClient", e);
            // для локалки это критичный косяк — падаем
            throw new IllegalStateException("Failed to configure SSL for local WebClient", e);
        }

        return WebClient.builder()
                .codecs(configure -> configure
                        .defaultCodecs()
                        .maxInMemorySize(Integer.parseInt(properties.getBufferSize())))
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .build();
    }
}
```
