```java
@Test
    void getAllMaterialTypes_whenBackendReturnsData_thenDelegatesToBackend() {
        // given
        when(properties.getSlugValueForMaterialType()).thenReturn("materialType");

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.TMC))
        ).thenReturn(resp);

        // when
        List<MaterialTypeDto> result = adapterCacheOps.getAllMaterialTypes();

        // then
        assertNotNull(result); // просто убеждаемся, что вызов отработал

        // проверяем, что правильно подготовлен запрос и вызван backend
        verify(properties).getSlugValueForMaterialType();

        ArgumentCaptor<ItemsSearchCriteriaRequest> requestCaptor =
                ArgumentCaptor.forClass(ItemsSearchCriteriaRequest.class);

        verify(baseMasterDataRequestService).requestData(
                requestCaptor.capture(),
                eq(SearchRequestProperties.Context.TMC)
        );

        ItemsSearchCriteriaRequest actualRequest = requestCaptor.getValue();
        assertEquals("materialType", actualRequest.getDictionaryName());
    }
```
