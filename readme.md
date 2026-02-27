```java/**
@Query("""
        select fx
        from FxRateEntity fx
        where fx.fromCurrencyCode = :fromCurrencyCode
          and fx.toCurrencyCode   = :toCurrencyCode
          and fx.useDate < :useDateExclusive
        order by fx.useDate desc
    """)
    List<FxRateEntity> findLatestRate(
            @Param("fromCurrencyCode") String fromCurrencyCode,
            @Param("toCurrencyCode") String toCurrencyCode,
            @Param("useDateExclusive") ZonedDateTime useDateExclusive,
            Pageable pageable
    );

    


```
