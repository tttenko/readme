```java
class StsTrackerValidatorsConfigTest {

    private final StsTrackerValidatorsConfig config = new StsTrackerValidatorsConfig();

    @Test
    void toApproveInValidator_shouldReturnTrue() {
        SwitchCheck validator = config.toApproveInValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of()));
    }

    @Test
    void approveInValidator_shouldReturnTrue() {
        SwitchCheck validator = config.approveInValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of()));
    }

    @Test
    void rejectInValidator_shouldReturnTrue() {
        SwitchCheck validator = config.rejectInValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of()));
    }

    @Test
    void toApproveOutValidator_shouldReturnTrue() {
        SwitchCheck validator = config.toApproveOutValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of()));
    }

    @Test
    void approveOutValidator_shouldReturnTrue() {
        SwitchCheck validator = config.approveOutValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of()));
    }

    @Test
    void rejectOutValidator_shouldReturnTrue() {
        SwitchCheck validator = config.rejectOutValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of()));
    }
}
```
