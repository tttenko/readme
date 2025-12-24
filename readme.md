```java
@Test
@DisplayName("test GET {host}/api/v1/info/country/{countryCode}")
void searchByAlpha2PathVariableTest() throws Exception {
    GetItemsSearchResponse response = reader.readResource(
            "mdresponse/country/country-RU.json",
            GetItemsSearchResponse.class
    );

    MvcTestUtils.mockPostResponse(
            "v1/items/byAttrValues",
            response,
            GetItemsSearchResponse.class,
            httpRequestHelper
    );

    MvcTestUtils.checkResult(
            MvcTestUtils.performGetOk(mockMvc, "/api/v1/info/country/RU"),
            1
    );
}
```
