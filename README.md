```java

Каскадное удаление нужно подтвердить для всей цепочки данных:

initiative_metric_type → initiative_metric_assignment → metric_applicability_request → metric_applicability_history.

То есть при удалении режима работы должны автоматически удаляться assignment, все связанные заявки и вся история по ним. Иначе удаление остановится на следующем FK.
```
