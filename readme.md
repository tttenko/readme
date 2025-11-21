```java
public static <T> void mockPostResponse(
        String url,
        Class<T> clazz,
        HttpRequestHelper httpRequestHelper,
        java.util.function.Predicate<String> bodyPredicate,
        T whenMatches,
        T otherwise) {

    when(httpRequestHelper.sendPostRequest(contains(url), anyString(), eq(clazz)))
        .thenAnswer(inv -> {
            String body = inv.getArgument(1);
            return bodyPredicate.test(body) ? whenMatches : otherwise;
        });
}

public static java.util.function.Predicate<String> hasBoolEquals(String attributeId, boolean value) {
    String id  = "\"attributeId\":\"" + attributeId + "\"";
    String op  = "\"operation\":\"BOOL_EQ\"";
    String val = "\"value\":[\"" + Boolean.toString(value) + "\"]";
    return body -> body.contains(id) && body.contains(op) && body.contains(val);
}

В requestDataImpl добавлен критерий BOOL_EQ по DZO только для словаря ТерБанков, фильтрация происходит на стороне master-data.

В application.yml заведены свойства attributeIdForTbDzo (UUID атрибута DZO) и dzoAttributeValue для конфигурирования фильтра

```
