```java
 @Test
    void getAllMaterialTypes_whenBackendResponseIsInvalid_thenThrowsAndCallsBackend() {
        // given
        when(properties.getSlugValueForMaterialType()).thenReturn("materialType");

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.TMC))
        ).thenReturn(resp);

        // when + then: реальный createResult(..) кидает MdaMdErrorResponseException
        assertThrows(
                MdaMdErrorResponseException.class,
                () -> adapterCacheOps.getAllMaterialTypes()
        );

        // и при этом мы проверяем, что сервис корректно сформировал запрос и дернул backend
        verify(properties).getSlugValueForMaterialType();

        ArgumentCaptor<ItemsSearchCriteriaRequest> requestCaptor =
                ArgumentCaptor.forClass(ItemsSearchCriteriaRequest.class);

        verify(baseMasterDataRequestService).requestData(
                requestCaptor.capture(),
                eq(SearchRequestProperties.Context.TMC)
        );

        ItemsSearchCriteriaRequest actualRequest = requestCaptor.getValue();
        assertEquals("materialType", actualRequest.getDictionaryName());

        verifyNoMoreInteractions(baseMasterDataRequestService, properties);
    }
```
