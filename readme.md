```java
public record ExchangeRateUpsertParams(
        String rquid,
        LocalDateTime rqtm,
        String subType,
        String code1,
        String isoNum1,
        String code2,
        String isoNum2,
        LocalDateTime useDate,
        BigDecimal lotSize,
        BigDecimal value
) {}

@Modifying
@Query(value = """
    INSERT INTO master_data_adapter.exchange_rate (
        rquid, rqtm, fxratesubtype, code1, isonum1, code2, isonum2, use_date, lot_size, value
    )
    VALUES (
        :#{#p.rquid}, :#{#p.rqtm}, :#{#p.subType}, :#{#p.code1}, :#{#p.isoNum1},
        :#{#p.code2}, :#{#p.isoNum2}, :#{#p.useDate}, :#{#p.lotSize}, :#{#p.value}
    )
    ON CONFLICT (fxratesubtype, isonum1, isonum2, use_date)
    DO UPDATE SET
        rquid = EXCLUDED.rquid,
        rqtm = EXCLUDED.rqtm,
        code1 = EXCLUDED.code1,
        code2 = EXCLUDED.code2,
        lot_size = EXCLUDED.lot_size,
        value = EXCLUDED.value
    """, nativeQuery = true)
int upsert(@Param("p") ExchangeRateUpsertParams p);
```
