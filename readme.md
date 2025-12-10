```java
static final int INITIAL_BUFFER_SIZE = 1024;

    /**
     * Создает и настраивает экземпляр ObjectMapper.
     */
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .configure(ALLOW_SINGLE_QUOTES, true)
                .configure(FAIL_ON_EMPTY_BEANS, false)
                .registerModules(new JavaTimeModule())
                .registerModules(new Jdk8Module())
                .enable(ACCEPT_EMPTY_STRING_AS_NULL_OBJECT)
                .disable(ADJUST_DATES_TO_CONTEXT_TIME_ZONE)
                .disable(FAIL_ON_UNKNOWN_PROPERTIES)
                .setSerializationInclusion(NON_NULL)
                .registerModule(new JavaTimeModule())
                .setSerializationInclusion(NON_EMPTY);
    }

    /**
     * Создает и возвращает экземпляр HttpRequestHelper, используя переданные зависимости.
     */
    @Bean
    public HttpRequestHelper prepareRequest(ObjectMapper mapper, WebClient webClient) {
        return new HttpRequestHelper(mapper, webClient);
    }

    /**
     * Общий метод расчета размера буфера.
     */
    static int getBufferSize(ConnectionProperties props) {
        if (isNotEmpty(props.getBufferSize())) {
            return INITIAL_BUFFER_SIZE * Integer.parseInt(props.getBufferSize());
        }
        return INITIAL_BUFFER_SIZE;
    }

```
