```java
@Test
@DisplayName("test GET {host}/api/v1/info/region_code/{regionCode}")
void searchByRegionCodePathVariableTest() throws Exception {
    GetItemsSearchResponse response = reader.readResource(
            "mdresponse/region/region-01.json",
            GetItemsSearchResponse.class
    );

    MvcTestUtils.mockPostResponse(
            "v1/items/byAttrValues",
            response,
            GetItemsSearchResponse.class,
            httpRequestHelper
    );

    MvcTestUtils.checkResult(
            MvcTestUtils.performGetOk(mockMvc, "/api/v1/info/region_code/05"),
            1
    );
}
```
