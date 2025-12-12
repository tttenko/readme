```java
@Test
    void masterDataExchangeStrategiesBeanCreated() {
        SearchRequestProperties properties = new SearchRequestProperties();
        properties.setBufferSize("2");

        ObjectMapper mapper = new ObjectMapper();

        ExchangeStrategies strategies = config.masterDataExchangeStrategies(properties, mapper);

        assertNotNull(strategies);
    }

    @Test
    void masterDataExchangeStrategiesConfiguresCborDecoderAndMaxInMemorySize() {
        SearchRequestProperties properties = new SearchRequestProperties();
        properties.setBufferSize("2");

        ObjectMapper mapper = new ObjectMapper();

        ExchangeStrategies strategies = config.masterDataExchangeStrategies(properties, mapper);

        List<Jackson2CborDecoder> cborDecoders = strategies.messageReaders().stream()
                .filter(r -> r instanceof DecoderHttpMessageReader)
                .map(r -> ((DecoderHttpMessageReader<?>) r).getDecoder())
                .filter(d -> d instanceof Jackson2CborDecoder)
                .map(d -> (Jackson2CborDecoder) d)
                .collect(Collectors.toList());

        assertFalse(cborDecoders.isEmpty(), "В ExchangeStrategies должен быть зарегистрирован Jackson2CborDecoder");

        Jackson2CborDecoder decoder = cborDecoders.get(0);
        assertSame(mapper, decoder.getObjectMapper(), "Должен использоваться переданный ObjectMapper");
        assertEquals(2, decoder.getMaxInMemorySize(), "maxInMemorySize должен браться из properties.getBufferSize()");
        assertTrue(decoder.canDecode(ResolvableType.forClass(Object.class), MediaType.APPLICATION_JSON),
                "Декодер должен уметь декодировать application/json");
    }

    @Test
    void masterDataExchangeStrategiesThrowsWhenBufferSizeIsNotNumber() {
        SearchRequestProperties properties = new SearchRequestProperties();
        properties.setBufferSize("not-a-number");

        ObjectMapper mapper = new ObjectMapper();

        assertThrows(NumberFormatException.class,
                () -> config.masterDataExchangeStrategies(properties, mapper));
    }
```
