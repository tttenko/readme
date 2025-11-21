```java
// полный набор (как сейчас в фикстуре)
    GetItemsSearchResponse full =
        reader.readResource("mdresponse/terbanks/tb-all.json", GetItemsSearchResponse.class);
    // новый файл с уже отфильтрованными по DZO=false ТБ (12 штук)
    GetItemsSearchResponse filtered =
        reader.readResource("mdresponse/terbanks/tb-all-dzo-false.json", GetItemsSearchResponse.class);

    MvcTestUtils.mockPostResponse(
        "/v1/items/byAttrValues",
        GetItemsSearchResponse.class,
        httpRequestHelper,
        MvcTestUtils.hasBoolEquals(
            searchRequestProperties.getAttributeIdForTbDzo(),
            searchRequestProperties.isDzoAttributeValue()
        ),
        filtered,   // когда в body есть критерий DZO=false
        full        // во всех остальных случаях
    );

    checkResult(performGetOk(mockMvc, "/api/v1/info/tb"), 12);
```
