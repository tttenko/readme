```java/**
class CalendarDateMapperTest {

    private CalendarDateMapper mapper;

    @BeforeEach
    void setUp() {
        mapper = new CalendarDateMapper(new ObjectMapper());
    }

    @Test
    void mapValuesToDto_whenDateTypeIsWork_thenSetsWorkAndDescription() {
        // given
        Map<String, Object> values = Map.of("slug", "2026-01-30");
        Map<String, Map<String, Object>> attributes = Map.of(
            "date", Map.of("value", "2026-01-30"),
            "dateType", dateTypeNode("4", "OK")
        );

        // when
        CalendarDateDto dto = mapper.mapValuesToDto(values, attributes);

        // then
        assertThat(dto.getDateShort()).isEqualTo("2026-01-30");
        assertThat(dto.getDayType()).isEqualTo(DayType.WORK);
        assertThat(dto.getDescription()).isEqualTo("OK");
    }

    @Test
    void mapValuesToDto_whenDateTypeMissing_thenDefaultsToCommonAndNullDescription() {
        // given
        Map<String, Object> values = Map.of("slug", "2026-01-30");
        Map<String, Map<String, Object>> attributes = Map.of(
            "date", Map.of("value", "2026-01-30")
            // dateType отсутствует
        );

        // when
        CalendarDateDto dto = mapper.mapValuesToDto(values, attributes);

        // then
        assertThat(dto.getDayType()).isEqualTo(DayType.COMMON);
        assertThat(dto.getDescription()).isNull();
    }

    @Test
    void mapValuesToDto_whenUnknownSlug_thenDefaultsToCommonButKeepsDescription() {
        // given
        Map<String, Object> values = Map.of("slug", "2026-01-30");
        Map<String, Map<String, Object>> attributes = Map.of(
            "date", Map.of("value", "2026-01-30"),
            "dateType", dateTypeNode("999", "Что-то")
        );

        // when
        CalendarDateDto dto = mapper.mapValuesToDto(values, attributes);

        // then
        assertThat(dto.getDayType()).isEqualTo(DayType.COMMON);
        assertThat(dto.getDescription()).isEqualTo("Что-то");
    }

    @Test
    void mapValuesToDto_whenSlugIsNullInTree_thenDefaultsToCommon() {
        // given
        Map<String, Object> values = Map.of("slug", "2026-01-30");
        Map<String, Map<String, Object>> attributes = Map.of(
            "date", Map.of("value", "2026-01-30"),
            "dateType", dateTypeNode(null, "Имя есть")
        );

        // when
        CalendarDateDto dto = mapper.mapValuesToDto(values, attributes);

        // then
        assertThat(dto.getDayType()).isEqualTo(DayType.COMMON);
        assertThat(dto.getDescription()).isEqualTo("Имя есть");
    }

    private static Map<String, Object> dateTypeNode(String slug, String name) {
        // под jsonPointer'ы:
        // "/valueRef/valueRefItem/item/slug"
        // "/valueRef/valueRefItem/item/name"
        return Map.of(
            "valueRef", Map.of(
                "valueRefItem", Map.of(
                    "item", Map.of(
                        "slug", slug,
                        "name", name
                    )
                )
            )
        );
    }
}
```
