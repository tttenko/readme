```java
@Test
void fetchByKeys_givenIds_delegatesToBackendWithSlugAndContext_andPropagatesException() {
    // given
    List<String> ids = List.of("MT1", "MT2");
    when(properties.getSlugValueForMaterialType()).thenReturn("materialType");

    var backendError = new MdaMdErrorResponseException(
            "msg", "description", "semantic"
    );

    // если бекенд падает — лоадер должен просто пробросить это исключение
    doThrow(backendError).when(baseMasterDataRequestService)
            .requestData(
                    eq("materialType"),
                    eq(ids),
                    eq(SearchRequestProperties.Context.TMC)
            );

    // when
    MdaMdErrorResponseException thrown =
            assertThrows(MdaMdErrorResponseException.class,
                    () -> loader.fetchByKeys(ids));

    // then
    assertSame(backendError, thrown); // именно тот же объект проброшен наружу

    verify(properties).getSlugValueForMaterialType();
    verify(baseMasterDataRequestService).requestData(
            eq("materialType"),
            eq(ids),
            eq(SearchRequestProperties.Context.TMC)
    );
    verifyNoMoreInteractions(baseMasterDataRequestService);
}
```
