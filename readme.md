```java/**
private static Map<String, Object> dateTypeNode(String slug, String name) {
    Map<String, Object> item = new java.util.HashMap<>();
    item.put("slug", slug);   // null OK
    item.put("name", name);   // null OK

    Map<String, Object> valueRefItem = new java.util.HashMap<>();
    valueRefItem.put("item", item);

    Map<String, Object> valueRef = new java.util.HashMap<>();
    valueRef.put("valueRefItem", valueRefItem);

    Map<String, Object> root = new java.util.HashMap<>();
    root.put("valueRef", valueRef);

    return root;
}
```
