```java/**
class XmlConfigTest {

    @Test
    void xmlMapperBean_shouldHaveExpectedSettings_andParseLocalDateTime() throws Exception {
        XmlMapper mapper = new XmlConfig().xmlMapper();

        assertThat(mapper.isEnabled(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)).isFalse();

        TestDto dto = mapper.readValue("<TestDto dt=\"2026-02-16T10:11:12\" unknown=\"x\"/>", TestDto.class);
        assertThat(dto.dt).isEqualTo(LocalDateTime.of(2026, 2, 16, 10, 11, 12));
    }

    static class TestDto {
        @com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlProperty(isAttribute = true, localName = "dt")
        public LocalDateTime dt;
    }
}

```
