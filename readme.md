```java/**
@Query(value = """
    SELECT *
    FROM exchange_rate
    WHERE from_currency_code = :fromCurrencyCode
      AND to_currency_code   = :toCurrencyCode
      AND use_date < :useDateExclusive
    ORDER BY use_date DESC
    LIMIT 1
    """, nativeQuery = true)
Optional<FxRateEntity> findLatestRateNative(
        @Param("fromCurrencyCode") String fromCurrencyCode,
        @Param("toCurrencyCode") String toCurrencyCode,
        @Param("useDateExclusive") ZonedDateTime useDateExclusive
);


```
