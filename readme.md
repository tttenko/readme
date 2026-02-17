```java
if (requestUid == null || requestUid.isBlank()) {
        throw new InvalidCurrencyRateXmlException(
                "Некорректный входной XML: отсутствует обязательный идентификатор запроса RqUID (элемент PutEODPriceNf/RqUID)."
        );
    }

    if (requestTime == null) {
        throw new InvalidCurrencyRateXmlException(
                "Некорректный входной XML: отсутствует обязательное время запроса RqTm (элемент PutEODPriceNf/RqTm) для RqUID=" + requestUid + "."
        );
    }
```
