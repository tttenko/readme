```java
@Test
void parseLenient_when32Hex_thenParsesUuid() {
    String raw = "91B68C835ECD364AD4EEB72F0BFA319A";
    UUID uuid = UuidUtils.parseLenient(raw);

    assertThat(uuid.toString()).isEqualTo("91b68c83-5ecd-364a-d4ee-b72f0bfa319a");
}

```
