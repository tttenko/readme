```java

@SuppressWarnings("unchecked")
private static Map<String, String> unwrapItemIfNeeded(Map<String, String> values) {
    // values на самом деле может быть {"item": {...}, "values": [...]}
    Map<String, Object> raw = (Map<String, Object>) (Map<?, ?>) values;

    Object maybeItem = raw.get(ITEM); // ITEM = "item"
    if (maybeItem instanceof Map) {
        return (Map<String, String>) maybeItem; // отдаем только item
    }

    // для старых словарей (тербанки и т.п.) ничего не меняем
    return values;
}

```
