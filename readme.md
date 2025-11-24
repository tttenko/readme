```java
@Test
void givenOnlyInn_whenSearchCounterpartiesByCriteria_thenBypassCacheAndCallLoaderByCriteria() {
    // given
    String inn = "7700000000";
    String kpp = null;

    Map<String, List<String>> criteria = Map.of(
            "innAttr", List.of(inn)
    );
    when(criteriaBuilder.buildCriteria(inn, kpp)).thenReturn(criteria);

    List<CounterpartyDto> directData = List.of(mock(CounterpartyDto.class));
    // loader теперь возвращает просто List<CounterpartyDto>
    when(loaderSupplierByCriteria.loadByCriteria(criteria)).thenReturn(directData);

    // when
    List<CounterpartyDto> result =
            supplierService.searchCounterpartiesByCriteria(inn, kpp);

    // then
    assertEquals(directData, result);

    verify(criteriaBuilder).buildCriteria(inn, kpp);
    verify(loaderSupplierByCriteria).loadByCriteria(criteria);

    verifyNoInteractions(cacheGetOrLoadService);
    verify(criteriaBuilder, never()).buildInnKppKey(any(), any());
}

```
