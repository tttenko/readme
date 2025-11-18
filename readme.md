```java

 @Test
void givenALLKey_whenFetch_thenUnionBucket() {
    String allKey = ALL_KEY;

    var itemA = NdsFullDto.builder()
            .id("1").name("N1").rate("5").code("A")
            .build();

    var itemB = NdsFullDto.builder()
            .id("2").name("N2").rate("10").code("B")
            .build();

    var items = List.of(itemA, itemB);
    GetItemsSearchResponse response = new GetItemsSearchResponse();

    try (MockedStatic<BaseMasterDataRequestService> st =
                 mockStatic(BaseMasterDataRequestService.class)) {

        when(baseService.requestData(any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.BOOK)))
                .thenReturn(response);

        st.when(() -> BaseMasterDataRequestService
                .createResultWithAttribute(response, ndsMapper))
                .thenReturn(items);

        // when
        List<NdsService2.RateBucket> buckets =
                loader.fetchByKeys(List.of(allKey));

        // then
        assertThat(buckets).hasSize(1);

        NdsService2.RateBucket allBucket = buckets.get(0);
        assertThat(allBucket.key()).isEqualTo(allKey);
        assertThat(allBucket.items())
                .extracting(NdsFullDto::getId)
                .containsExactlyInAnyOrder("1", "2");

        verify(baseService, times(1)).requestData(
                any(ItemsSearchCriteriaRequest.class),
                eq(SearchRequestProperties.Context.BOOK)
        );
    }
}
```
