```java
@Slf4j
public final class MasterDataWebClientFilters {

    private MasterDataWebClientFilters() {}

    /** Один фильтр: лог + единый маппинг ошибок в Mda*Exception */
    public static ExchangeFilterFunction md() {
        return (req, next) -> {
            log.info("MD -> {} {}", req.method(), req.url());

            return next.exchange(req)
                    .flatMap(resp -> {
                        log.info("MD <- {} {}", resp.statusCode().value(), req.url());

                        if (resp.statusCode().is2xxSuccessful()) {
                            return Mono.just(resp);
                        }

                        // прочитать body и получить стандартный WebClientResponseException
                        return resp.createException()
                                .flatMap(ex -> Mono.error(mapToMda((WebClientResponseException) ex)));
                    })
                    .onErrorMap(DataBufferLimitException.class,
                            e -> new MdaDataBufferLimitException(e.getMessage()))
                    .onErrorMap(WebClientResponseException.class,
                            MasterDataWebClientFilters::mapToMda);
        };
    }

    private static RuntimeException mapToMda(WebClientResponseException ex) {
        if (ex.getCause() instanceof DataBufferLimitException) {
            return new MdaDataBufferLimitException(ex.getMessage());
        }
        if (ex.getStatusCode().value() == 500) {
            return new MdaInternalMdServerException(ex.getMessage());
        }

        String body = ex.getResponseBodyAsString();
        if (body == null) body = "";

        return new MdaServerErrorException(
                ex.getMessage(),
                ex.getStatusCode().value(),
                "Ошибка сервера",
                body
        );
    }
}
```
