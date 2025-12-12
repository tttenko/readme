```java
@Test
    void strategiesUseConfiguredObjectMapper_singleQuotesAreParsed() {
        CommonApplicationConfig cfg = new CommonApplicationConfig();

        SearchRequestProperties props = new SearchRequestProperties();
        props.setBufferSize("1024");

        ObjectMapper mapper = cfg.objectMapper();
        ExchangeStrategies strategies = cfg.masterDataExchangeStrategies(props, mapper);

        Jackson2JsonDecoder decoder = strategies.messageReaders().stream()
                .filter(DecoderHttpMessageReader.class::isInstance)
                .map(DecoderHttpMessageReader.class::cast)
                .map(r -> r.getDecoder())
                .filter(Jackson2JsonDecoder.class::isInstance)
                .map(Jackson2JsonDecoder.class::cast)
                .findFirst()
                .orElseThrow();

        var buf = new DefaultDataBufferFactory()
                .wrap("{'x':1}".getBytes(StandardCharsets.UTF_8));

        @SuppressWarnings("unchecked")
        Map<String, Object> result = (Map<String, Object>) decoder.decodeToMono(
                Mono.just(buf),
                ResolvableType.forClass(Map.class),
                MediaType.APPLICATION_JSON,
                Map.of()
        ).block();

        assertEquals(1, ((Number) result.get("x")).intValue());
    }



```
