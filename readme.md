```java
public static Predicate<String> hasBoolEquals(UUID attrId, boolean value) {
    String id = attrId.toString();
    String val = Boolean.toString(value); // "true"/"false"
    return body ->
            body.contains("\"attributeId\":\"" + id + "\"") &&
            body.contains("\"operation\":\"BOOL_EQ\"") &&
            // на всякий случай допускаем оба варианта сериализации
            (body.contains("\"value\":[" + val + "]") ||
             body.contains("\"value\":[\"" + val + "\"]"));
}
```
