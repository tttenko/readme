```java
if (requestTime == null) {
        throw new InvalidCurrencyRateXmlException(String.format(
                "Некорректный входной XML: отсутствует обязательное время запроса RqTm (элемент PutEODPriceNf/RqTm). RqUID=%s.",
                requestUid
        ));
    }
```
