```java
public enum DaysClassification {
    ALL("all"),
    WORK("work");

    private final String wire;

    DaysClassification(String wire) {
        this.wire = wire;
    }

    public static DaysClassification parseOrDefault(String raw) {
        if (raw == null || raw.isBlank()) return ALL;
        String v = raw.trim().toLowerCase();

        for (DaysClassification dc : values()) {
            if (dc.wire.equals(v)) return dc;
        }
        throw new IllegalArgumentException("daysClassification must be one of: all, work");
    }
}
```
