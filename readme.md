```java
@Test
@DisplayName("тест GET {host}/api/v1/info/tb")
void searchTbAllTest() throws Exception {
    // 1. Читаем оба ответа из ресурсов
    GetItemsSearchResponse full =
            reader.readResource("mdresponce/terbanks/tb-all.json", GetItemsSearchResponse.class);
    GetItemsSearchResponse filtered =
            reader.readResource("mdresponce/terbanks/tb-all-dzo-false.json", GetItemsSearchResponse.class);

    // 2. Мокаем master-data
    lenient().when(httpRequestHelper.sendPostRequest(
                    contains("/v1/items/byAttrValues"),   // как и в старом mockPostResponse
                    anyString(),
                    eq(GetItemsSearchResponse.class)
            ))
            .thenAnswer(invocation -> {
                String body = invocation.getArgument(1, String.class);

                // здесь проверяем, что в body реально прилетел фильтр по DZO
                String dzoAttrId = searchRequestProperties.getAttributeIdForTbDzo();
                String dzoVal = Boolean.toString(searchRequestProperties.isDzoAttributeValue()); // "false"

                boolean hasDzoFalse =
                        body.contains("\"attributeId\":\"" + dzoAttrId + "\"") &&
                        body.contains("\"operation\":\"BOOL_EQ\"") &&
                        (
                             body.contains("\"value\":[" + dzoVal + "]") ||
                             body.contains("\"value\":[\"" + dzoVal + "\"]")
                        );

                // если в запросе есть фильтр DZO=false – возвращаем отфильтрованный json
                return hasDzoFalse ? filtered : full;
            });

    // 3. Делаем запрос к контроллеру и проверяем, что вернулось 12 записей
    checkResult(performGetOk(mockMvc, "/api/v1/info/tb"), 12);
}
```
