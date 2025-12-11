```java
 class RequestFactoryTest {

    // Используем тот же ObjectMapper, что и в приложении
    private final ObjectMapper mapper = new CommonApplicationConfig().objectMapper();

    @Test
    void getRequestType1() throws JsonProcessingException {
        String expectedJson = """
        {
          "countItemsOnly" : "false",
          "returnItemsWithAttributes" : "true",
          "reference" : {
            "slug" : "TMC_ROOT"
          },
          "searchCriteriaItems" : [ {
            "itemFieldName" : "slug",
            "searchConditions" : [ {
              "value" : [ "000000000020033840" ],
              "operation" : "EQ"
            } ]
          } ]
        }
        """;

        var request = RequestFactory.getByAttrValuesBuilder()
                .withAttributes(true)
                .dictionaryName("TMC_ROOT")
                .addItemFieldName("slug", "000000000020033840")
                .build();

        JsonNode json1 = mapper.readTree(expectedJson);
        JsonNode json2 = mapper.readTree(mapper.writeValueAsString(request));

        assertEquals(json1, json2);
    }

    @Test
    void getRequestType2() throws JsonProcessingException {
        String expectedJson = """
        {
          "countItemsOnly" : "false",
          "returnItemsWithAttributes" : "false",
          "reference" : {
            "slug" : "VID_TMC_ROOT"
          },
          "searchCriteriaItems" : [ {
            "itemFieldName" : "slug",
            "searchConditions" : [ {
              "value" : [ "0000293887474" ],
              "operation" : "EQ"
            } ]
          } ]
        }
        """;

        var request = RequestFactory.getByAttrValuesBuilder()
                .withAttributes(false)
                .dictionaryName("VID_TMC_ROOT")
                .addItemFieldName("slug", "0000293887474")
                .build();

        JsonNode json1 = mapper.readTree(expectedJson);
        JsonNode json2 = mapper.readTree(mapper.writeValueAsString(request));

        assertEquals(json1, json2);
    }

    @Test
    void getRequestType3() throws JsonProcessingException {
        String expectedJson = """
        {
          "countItemsOnly" : "false",
          "returnItemsWithAttributes" : "true",
          "reference" : {
            "slug" : "be"
          },
          "searchCriteriaItems" : [ {
            "itemFieldName" : "slug",
            "searchConditions" : [ {
              "value" : [ "5500" ],
              "operation" : "EQ"
            } ]
          } ]
        }
        """;

        var request = RequestFactory.getByAttrValuesBuilder()
                .withAttributes(true)
                .dictionaryName("be")
                .addItemFieldName("slug", "5500")
                .build();

        JsonNode json1 = mapper.readTree(expectedJson);
        JsonNode json2 = mapper.readTree(mapper.writeValueAsString(request));

        assertEquals(json1, json2);
    }

    @Test
    void getRequestType4() throws JsonProcessingException {
        String expectedJson = """
        {
          "countItemsOnly" : "false",
          "returnItemsWithAttributes" : "true",
          "reference" : {
            "slug" : "DA_partner"
          },
          "pagination" : {
            "limit" : 20
          }
        }
        """;

        var request = RequestFactory.getByAttrValuesBuilder()
                .withAttributes(true)
                .dictionaryName("DA_partner")
                .limit(20)
                .build();

        JsonNode json1 = mapper.readTree(expectedJson);
        JsonNode json2 = mapper.readTree(mapper.writeValueAsString(request));

        assertEquals(json1, json2);
    }

    @Test
    void getRequestType5() throws JsonProcessingException {
        String expectedJson = """
        {
          "countItemsOnly" : "false",
          "returnItemsWithAttributes" : "true",
          "reference" : {
            "slug" : "DA_partner"
          },
          "searchCriteria" : [ {
            "attributeId" : "518a6897-40b8-4113-aa83-05c8c07deca0",
            "searchConditions" : [ {
              "value" : [ "0713200099" ],
              "operation" : "CONT"
            } ]
          } ],
          "pagination" : {
            "limit" : 20
          }
        }
        """;

        var request = RequestFactory.getByAttrValuesBuilder()
                .withAttributes(true)
                .dictionaryName("DA_partner")
                .addAttributesAndValue(
                        "518a6897-40b8-4113-aa83-05c8c07deca0",
                        Pair.of(List.of("0713200099"), "CONT")
                )
                .limit(20)
                .build();

        JsonNode json1 = mapper.readTree(expectedJson);
        JsonNode json2 = mapper.readTree(mapper.writeValueAsString(request));

        assertEquals(json1, json2);
    }

    @Test
    void getRequestType6() throws JsonProcessingException {
        String expectedJson = """
        {
          "countItemsOnly" : "false",
          "returnItemsWithAttributes" : "true",
          "reference" : {
            "slug" : "pro_doc_currency"
          },
          "searchCriteria" : [ {
            "attributeId" : "9a87f628-9473-436c-9930-985476c5e2e1",
            "searchConditions" : [ {
              "value" : [ "USD" ],
              "operation" : "EQ"
            } ]
          } ]
        }
        """;

        var request = RequestFactory.getByAttrValuesBuilder()
                .withAttributes(true)
                .dictionaryName("pro_doc_currency")
                .addAttributesAndValue(
                        "9a87f628-9473-436c-9930-985476c5e2e1",
                        Pair.of(List.of("USD"), "EQ")
                )
                .build();

        JsonNode json1 = mapper.readTree(expectedJson);
        JsonNode json2 = mapper.readTree(mapper.writeValueAsString(request));

        assertEquals(json1, json2);
    }

    @Test
    void getRequestType7() throws JsonProcessingException {
        String expectedJson = """
        {
          "countItemsOnly" : "false",
          "returnItemsWithAttributes" : "true",
          "reference" : {
            "slug" : "DA_partnerAccount"
          },
          "searchCriteria" : [ {
            "attributeId" : "101a5ac4-97fe-4171-af2d-64d5f0516955",
            "searchConditions" : [ {
              "operation" : "EQ",
              "refItemSlug" : [ "42002617" ]
            } ]
          } ],
          "pagination" : {
            "limit" : 20,
            "offset" : 100
          }
        }
        """;

        var request = RequestFactory.getByAttrValuesBuilder()
                .withAttributes(true)
                .dictionaryName("DA_partnerAccount")
                .addAttributesAndRefItemSlug(
                        "101a5ac4-97fe-4171-af2d-64d5f0516955",
                        "42002617"
                )
                .limit(20)
                .offset(100)
                .build();

        JsonNode json1 = mapper.readTree(expectedJson);
        JsonNode json2 = mapper.readTree(mapper.writeValueAsString(request));

        assertEquals(json1, json2);
    }

    @Test
    void getRequestType8() throws JsonProcessingException {
        String expectedJson = """
        {
          "reference" : {
            "slug" : "DA_partner",
            "returnItemsWithAttributes" : "true"
          },
          "slugs" : [ "23000391" ]
        }
        """;

        var request = RequestFactory.getItemsBySlugsBuilder()
                .withAttributes(true)
                .dictionaryName("DA_partner")
                .withSlugs(List.of("23000391"))
                .build();

        JsonNode json1 = mapper.readTree(expectedJson);
        JsonNode json2 = mapper.readTree(mapper.writeValueAsString(request));

        assertEquals(json1, json2);
    }
}
```
