```java/**
private String stripBom(String xml) {
    String s = xml == null ? "" : xml.strip();
    return (!s.isEmpty() && s.charAt(0) == '\uFEFF') ? s.substring(1).strip() : s;
}
```
