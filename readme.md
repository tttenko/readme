```java
@Test
void parseLenient_when32Hex_thenParsesUuid() {
    String raw = "91B68C835ECD364AD4EEB72F0BFA319A";
    UUID uuid = UuidUtils.parseLenient(raw);

    assertThat(uuid.toString()).isEqualTo("91b68c83-5ecd-364a-d4ee-b72f0bfa319a");
}

private UuidUtils() {}

    public static UUID parseLenient(String raw) {
        if (raw == null) return null;
        String s = raw.trim();
        if (s.isEmpty()) return null;

        // 32 hex без дефисов -> вставляем дефисы
        if (s.matches("(?i)^[0-9a-f]{32}$")) {
            s = s.substring(0, 8) + "-" +
                s.substring(8, 12) + "-" +
                s.substring(12, 16) + "-" +
                s.substring(16, 20) + "-" +
                s.substring(20);
        }

        return UUID.fromString(s);
    }

```
