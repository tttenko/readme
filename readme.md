```java
@Log4j2
@Component
@RequiredArgsConstructor
public class BaseMasterDataRequestService {

    protected static final String SUCCESS = "S";

    // это твои конфиги/лимиты/атрибуты — оставить как есть
    public final SearchRequestProperties properties;

    @Qualifier("mdBookApi")
    private final ApiApi mdBookApi;

    @Qualifier("mdTmcApi")
    private final ApiApi mdTmcApi;

    public GetItemsSearchResponse requestData(ItemsSearchCriteriaRequest request,
                                              SearchRequestProperties.Context context) {
        try {
            ResponseEntity<GetItemsSearchResponse> resp =
                    api(context).findItemsByAttributeValues(request);

            // Если вдруг вернулся не 2xx (в зависимости от настроек адаптера)
            if (!resp.getStatusCode().is2xxSuccessful()) {
                throw new MdaServerErrorException(
                        "MD call failed: HTTP " + resp.getStatusCode(),
                        resp.getStatusCode().value(),
                        "Ошибка сервера",
                        null
                );
            }

            GetItemsSearchResponse body = resp.getBody();
            if (body == null) {
                throw new MdaServerErrorException(
                        "MD call failed: empty body",
                        502,
                        "Пустой ответ от МД",
                        null
                );
            }

            return body;

        } catch (WebClientResponseException ex) {
            // 4xx/5xx если адаптер выбрасывает исключение
            if (ex.getCause() instanceof DataBufferLimitException) {
                throw new MdaDataBufferLimitException(ex.getMessage());
            }
            if (ex instanceof WebClientResponseException.InternalServerError) {
                throw new MdaInternalMdServerException(ex.getMessage());
            }
            throw new MdaServerErrorException(
                    ex.getMessage(),
                    ex.getStatusCode().value(),
                    "Ошибка сервера",
                    null
            );
        } catch (Exception ex) {
            // на всякий случай сохраним поведение “как раньше”
            if (ex.getCause() instanceof DataBufferLimitException) {
                throw new MdaDataBufferLimitException(ex.getMessage());
            }
            throw ex;
        }
    }

    private ApiApi api(SearchRequestProperties.Context context) {
        return context == SearchRequestProperties.Context.TMC ? mdTmcApi : mdBookApi;
    }





@Configuration
@RequiredArgsConstructor
public class MasterDataApiClientConfig {

    @Qualifier("masterDataWebClient")
    private final WebClient webClient;

    private final SearchRequestProperties props;

    @Bean("mdBookApi")
    public ApiApi mdBookApi() {
        return createApi(joinBaseUrl(props.getUrl(), props.getBookContext()));
    }

    @Bean("mdTmcApi")
    public ApiApi mdTmcApi() {
        return createApi(joinBaseUrl(props.getUrl(), props.getTmcContext()));
    }

    private ApiApi createApi(String baseUrl) {
        WebClient client = webClient.mutate()
                .baseUrl(baseUrl)
                .build();

        HttpServiceProxyFactory factory = HttpServiceProxyFactory
                .builderFor(WebClientAdapter.create(client))
                .build();

        return factory.createClient(ApiApi.class);
    }

    private static String joinBaseUrl(String url, String context) {
        String u = url == null ? "" : url.replaceAll("/+$", "");
        String c = context == null ? "" : context.replaceAll("^/+", "").replaceAll("/+$", "");
        return u + "/" + c;
    }
}

```
