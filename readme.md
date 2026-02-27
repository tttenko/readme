```java/**
class DayTypeTest {

    @Test
    void givenNullRawValue_whenParseOrDefault_thenReturnsCommon() {
        // given
        String rawValue = null;

        // when
        DayType result = DayType.parseOrDefault(rawValue);

        // then
        assertThat(result).isEqualTo(DayType.COMMON);
    }

    @Test
    void givenBlankRawValue_whenParseOrDefault_thenReturnsCommon() {
        // given
        String rawValue = "   ";

        // when
        DayType result = DayType.parseOrDefault(rawValue);

        // then
        assertThat(result).isEqualTo(DayType.COMMON);
    }

    @Test
    void givenValueWithSpaces_whenParseOrDefault_thenStripsAndParses() {
        // given
        String rawValue = " 4 ";

        // when
        DayType result = DayType.parseOrDefault(rawValue);

        // then
        assertThat(result).isEqualTo(DayType.WORK);
    }

    @Test
    void givenCommonValue_whenParseOrDefault_thenReturnsCommon() {
        // given
        String rawValue = "1";

        // when
        DayType result = DayType.parseOrDefault(rawValue);

        // then
        assertThat(result).isEqualTo(DayType.COMMON);
    }

    @Test
    void givenUndefinedValue_whenParseOrDefault_thenReturnsUndefined() {
        // given
        String rawValue = "0";

        // when
        DayType result = DayType.parseOrDefault(rawValue);

        // then
        assertThat(result).isEqualTo(DayType.UNDEFINED);
    }

    @Test
    void givenUnknownValue_whenParseOrDefault_thenReturnsCommon() {
        // given
        String rawValue = "999";

        // when
        DayType result = DayType.parseOrDefault(rawValue);

        // then
        assertThat(result).isEqualTo(DayType.COMMON);
    }

    @Test
    void givenNonNumericValue_whenParseOrDefault_thenReturnsCommon() {
        // given
        String rawValue = "WORK";

        // when
        DayType result = DayType.parseOrDefault(rawValue);

        // then
        assertThat(result).isEqualTo(DayType.COMMON);
    }
}
```
