```java
/**
     * WebClient с использованием сертификата (бывшая if-ветка).
     */
    @Bean
    public WebClient webClient(ConnectionProperties props) {
        if (isNotEmpty(props.getFilePathTrustStore())) {
            ClassPathResource resource = new ClassPathResource(props.getFilePathKeyStore());
            HttpClient httpClient = HttpClient.create();

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
                log.error(e.getMessage(), e);
            }

            return WebClient.builder()
                    .codecs(configure -> configure
                            .defaultCodecs()
                            .maxInMemorySize(getBufferSize(props)))
                    .clientConnector(new ReactorClientHttpConnector(httpClient))
                    .build();
        }

        // fallback если вдруг сертификат не задан
        return WebClient.builder()
                .codecs(configure -> configure
                        .defaultCodecs()
                        .maxInMemorySize(getBufferSize(props)))
                .build();
    }
}

```
