```java
@Test
    void getAllMaterialTypes_whenBackendReturnsData_thenMapsResult() {
        // given
        when(properties.getSlugValueForMaterialType()).thenReturn("materialType");

        GetItemsSearchResponse resp = new GetItemsSearchResponse();
        when(baseMasterDataRequestService.requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.TMC))
        ).thenReturn(resp);

        List<MaterialTypeDto> mapped = List.of(
                MaterialTypeDto.builder().typeId("TYPE1").build()
        );

        try (MockedStatic<BaseMasterDataRequestService> statics =
                     mockStatic(BaseMasterDataRequestService.class)) {

            @SuppressWarnings("unchecked")
            DataMapper<MaterialTypeDto> mapper =
                    (DataMapper<MaterialTypeDto>) materialTypeMapper;

            statics.when(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, mapper)
            ).thenReturn(mapped);

            // when
            List<MaterialTypeDto> result = adapterCacheOps.getAllMaterialTypes();

            // then
            assertThat(result).containsExactlyElementsOf(mapped);

            verify(properties).getSlugValueForMaterialType();
            verify(baseMasterDataRequestService).requestData(
                    any(ItemsSearchCriteriaRequest.class),
                    eq(SearchRequestProperties.Context.TMC)
            );
            statics.verify(() ->
                    BaseMasterDataRequestService.createResultWithAttribute(resp, mapper)
            );
        }
    }
```
