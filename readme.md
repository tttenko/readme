```java

Согласен, по сигнатуре тут Map<String, String>, но из BaseMasterDataRequestService.createResult фактически прилетает raw-Map вида {"item": {...}, "values": [...]} — тип подрезан до Map<String, String> ради совместимости с остальными мапперами. Я как раз и делаю unwrap по ключу item, чтобы вытащить реальные поля (name/slug) и при этом не переписывать общий сервис.

```
