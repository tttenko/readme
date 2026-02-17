```java/**
 * Параметры для UPSERT записи курса в таблицу {@code exchange_rate} (вставка или обновление по бизнес-ключу).
 */
public record ExchangeRateUpsertParams(
        String rquid,
        ZonedDateTime rgtm,
        String subType,
        String code1,
        String isoNum1,
        String code2,
        String isoNum2,
        ZonedDateTime useDate,
        BigDecimal lotSize,
        BigDecimal value
) {}
```
