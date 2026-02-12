```java
public String resolve(ConsumerRecord<String, String> record) {
    String root = FxRateXmlSupport.extractRoot(FxRateXmlSupport.stripBom(record.value()));
    if (root == null || root.isBlank() || "UNKNOWN".equals(root)) {
        throw new IllegalArgumentException("Cannot resolve routeKey from XML root tag");
    }
    return root;
}
```
