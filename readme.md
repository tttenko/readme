```java
public final class FxRateXmlSupport {

    private static final XMLInputFactory FACTORY = XMLInputFactory.newFactory();

    private FxRateXmlSupport() {}

    public static String stripBom(String s) {
        if (s == null) return "";
        String t = s.trim();
        return t.startsWith("\uFEFF") ? t.substring(1).trim() : t;
    }

    public static String rootTag(String xml) {
        String s = stripBom(xml);
        if (s.isBlank()) return "UNKNOWN";

        try (StringReader sr = new StringReader(s)) {
            XMLStreamReader r = FACTORY.createXMLStreamReader(sr);
            try {
                while (r.hasNext()) {
                    if (r.next() == XMLStreamConstants.START_ELEMENT) {
                        return r.getLocalName(); // PutEODPriceNf
                    }
                }
                return "UNKNOWN";
            } finally {
                try { r.close(); } catch (Exception ignore) {}
            }
        } catch (Exception e) {
            throw new IllegalArgumentException("Cannot read XML root tag", e);
        }
    }
}
```
