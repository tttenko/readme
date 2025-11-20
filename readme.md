```java
@Test
void fetchByKeys_givenIds_delegatesToBackendWithSlugAndContext_andPropagatesException() {
    // given
    List<String> ids = List.of("MT1", "MT2");
    when(properties.getSlugValueForMaterialType()).thenReturn("materialType");

    var backendError = new MdaMdErrorResponseException(
            "msg", "description", "semantic"
    );

    // если мок всё же срабатывает – пусть кидает именно наш кастомный эксепшн
    doThrow(backendError).when(baseMasterDataRequestService)
            .requestDataWithAttribute(
                    eq("materialType"),
                    eq(ids),
                    eq(SearchRequestProperties.Context.TMC)
            );

    // when
    MdaMdErrorResponseException thrown =
            assertThrows(MdaMdErrorResponseException.class,
                    () -> loader.fetchByKeys(ids));

    // then
    // нам важно, что наружу вылетает именно кастомное исключение
    // (не обязательно тот же самый объект, поэтому assertSame не нужен)
    assertThat(thrown).isInstanceOf(MdaMdErrorResponseException.class);

    verify(properties).getSlugValueForMaterialType();
    verify(baseMasterDataRequestService).requestDataWithAttribute(
            eq("materialType"),
            eq(ids),
            eq(SearchRequestProperties.Context.TMC)
    );
    verifyNoMoreInteractions(baseMasterDataRequestService);
}
```
