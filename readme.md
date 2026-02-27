```java/**
class DayTypeTest {

    @ParameterizedTest(name = "rawValue={0} -> {1}")
    @MethodSource("parseOrDefaultCases")
    void parseOrDefault_shouldReturnExpectedDayType(String rawValue, DayType expected) {
        DayType result = DayType.parseOrDefault(rawValue);
        assertThat(result).isEqualTo(expected);
    }

    static Stream<Arguments> parseOrDefaultCases() {
        return Stream.of(
            // default branch: !hasText(rawValue) => COMMON
            Arguments.of(null, DayType.COMMON),
            Arguments.of("", DayType.COMMON),
            Arguments.of("   ", DayType.COMMON),
            Arguments.of("\t", DayType.COMMON),

            // happy-path: matches by internal "value"
            Arguments.of("1", DayType.COMMON),
            Arguments.of(" 1 ", DayType.COMMON),
            Arguments.of("4", DayType.WORK),
            Arguments.of(" 4 ", DayType.WORK),
            Arguments.of("0", DayType.UNDEFINED),
            Arguments.of(" 0 ", DayType.UNDEFINED),

            // fallback: unknown => COMMON
            Arguments.of("999", DayType.COMMON),
            Arguments.of("WORK", DayType.COMMON),
            Arguments.of("work", DayType.COMMON)
        );
    }
}
```
