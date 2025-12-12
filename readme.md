```java
int maxBytes = Integer.parseInt(props.getBufferSize());

        // JSON decoder/encoder строго с твоим ObjectMapper
        Jackson2JsonDecoder decoder = new Jackson2JsonDecoder(objectMapper, MediaType.APPLICATION_JSON);
        decoder.setMaxInMemorySize(maxBytes);

        Jackson2JsonEncoder encoder = new Jackson2JsonEncoder(objectMapper, MediaType.APPLICATION_JSON);

        return ExchangeStrategies.builder()
                .codecs(c -> {
                    c.registerDefaults(false);              // <-- ключевой момент: не поднимать дефолты (и CBOR тоже)
                    c.customCodecs().register(decoder);
                    c.customCodecs().register(encoder);
                })
                .build();



```
