```java
@Cacheable(cacheNames = TerBankService_2.TB_ALL, key = "'ALL'", sync = true)
  @Nonnull
  public List<TerBankDto> getAllBanks() {
    var resp = requestDataWithAttribute(properties.getSlugValueForTerBank(), null, SearchRequestProperties.Context.BOOK);
    return createResult(resp, terBankMapper);
  }

  @Cacheable(cacheNames = TerBankService_2.TB_REQ_ALL, key = "'ALL'", sync = true)
  @Nonnull
  public List<TerBankWithRequisiteDto> getAllBanksWithRequisite() {
    var resp = requestDataWithAttribute(properties.getSlugValueForTerBank(), null, SearchRequestProperties.Context.BOOK);
    return createWithAttribute(resp, terBankWithRequisiteMapper);
  }

```
