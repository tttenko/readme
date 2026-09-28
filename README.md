```java

Если заявка PENDING и is_visible_in_office=true, то у соответствующего initiative_metric_assignment статус должен быть PENDING. Если при этом метрика остаётся ACTIVE — состояние БД неконсистентное.
Если же заявка PENDING, но is_visible_in_office=false, это отменённая координатором заявка после restore, и тогда assignment=ACTIVE — ожидаемое состояние.
Если заявка была вставлена вручную через БД, статус assignment автоматически не поменяется — штатный POST меняет и request, и assignment одной транзакцией.
```
