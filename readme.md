```java
@Component
@RequiredArgsConstructor
public class MasterDataApiApiClient implements MasterDataClient {

    @Qualifier("mdBookApi")
    private final ApiApi mdBookApi;

    @Qualifier("mdTmcApi")
    private final ApiApi mdTmcApi;

    @Override
    public GetItemsSearchResponse findItemsByAttributeValues(
            ItemsSearchCriteriaRequest request,
            SearchRequestProperties.Context context
    ) {
        ApiApi api = (context == SearchRequestProperties.Context.TMC) ? mdTmcApi : mdBookApi;

        ResponseEntity<GetItemsSearchResponse> resp = api.findItemsByAttributeValues(request);

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
                    "Empty body from MD",
                    502,
                    "Пустой ответ от МД",
                    null
            );
        }

        return body;
    }
}
```
