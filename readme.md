```java
if (rquid == null) throw new IllegalArgumentException("Missing RQUID");
  if (rqtm == null) throw new IllegalArgumentException("Missing RqTm");

  List<FxRateXmlDto> rates = message.getFxRates();
  if (rates == null || rates.isEmpty()) return 0;

  int processed = 0;

  for (FxRateXmlDto x : rates) {
    if (x == null) continue;

    String subType = x.fxRateSubType;
    String code1 = x.code1;
    String code2 = x.code2;
    LocalDate useDate = x.useDate;

    // ключевые поля: если чего-то нет/не распарсилось -> skip только эту запись
    if (subType == null || code1 == null || code2 == null || useDate == null) {
      // log.warn("Skip FXRate with missing key. rquid={}, subType={}, code1={}, code2={}, useDate={}",
      //          rquid, subType, code1, code2, useDate);
      continue;
    }

    BigDecimal lotSize = x.lotSize;
    BigDecimal value = x.value;

    // если число не распарсилось -> null -> skip только эту запись
    if (lotSize == null || value == null) {
      // log.warn("Skip FXRate with invalid numbers. rquid={}, code1={}, code2={}, useDate={}, lotSize={}, value={}",
      //          rquid, code1, code2, useDate, lotSize, value);
      continue;
    }

    // lotSize=0 — это не причина валить весь документ
    if (lotSize.compareTo(BigDecimal.ZERO) == 0) {
      // log.warn("Skip FXRate with lotSize=0. rquid={}, code1={}, code2={}, useDate={}", rquid, code1, code2, useDate);
      continue;
    }

    repo.upsert(
      rquid, rqtm,
      subType,
      code1, x.isoNum1,
      code2, x.isoNum2,
      useDate,
      lotSize, value,
      x.isPublic
    );

    processed++;
  }

  return processed;
}
```
