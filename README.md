```java
class StsTrackerValidatorsConfigTest {

    private final StsTrackerValidatorsConfig config = new StsTrackerValidatorsConfig();

    @Test
    void toApproveInValidator_shouldReturnTrue_whenActionIsToApprove() {
        SwitchCheck validator = config.toApproveInValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of("action", StsAction.TO_APPROVE.getId())));
    }

    @Test
    void toApproveInValidator_shouldReturnFalse_whenActionIsNotToApprove() {
        SwitchCheck validator = config.toApproveInValidator();

        assertNotNull(validator);
        assertFalse(validator.check(null, null, Map.of("action", StsAction.APPROVE.getId())));
    }

    @Test
    void approveInValidator_shouldReturnTrue_whenActionIsApprove() {
        SwitchCheck validator = config.approveInValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of("action", StsAction.APPROVE.getId())));
    }

    @Test
    void approveInValidator_shouldReturnFalse_whenActionIsNotApprove() {
        SwitchCheck validator = config.approveInValidator();

        assertNotNull(validator);
        assertFalse(validator.check(null, null, Map.of("action", StsAction.REJECT.getId())));
    }

    @Test
    void rejectInValidator_shouldReturnTrue_whenActionIsReject() {
        SwitchCheck validator = config.rejectInValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of("action", StsAction.REJECT.getId())));
    }

    @Test
    void rejectInValidator_shouldReturnFalse_whenActionIsNotReject() {
        SwitchCheck validator = config.rejectInValidator();

        assertNotNull(validator);
        assertFalse(validator.check(null, null, Map.of("action", StsAction.APPROVE.getId())));
    }

    @Test
    void toApproveOutValidator_shouldReturnTrue_whenActionIsToApprove() {
        SwitchCheck validator = config.toApproveOutValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of("action", StsAction.TO_APPROVE.getId())));
    }

    @Test
    void toApproveOutValidator_shouldReturnFalse_whenActionIsNotToApprove() {
        SwitchCheck validator = config.toApproveOutValidator();

        assertNotNull(validator);
        assertFalse(validator.check(null, null, Map.of("action", StsAction.REJECT.getId())));
    }

    @Test
    void approveOutValidator_shouldReturnTrue_whenActionIsApprove() {
        SwitchCheck validator = config.approveOutValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of("action", StsAction.APPROVE.getId())));
    }

    @Test
    void approveOutValidator_shouldReturnFalse_whenActionIsNotApprove() {
        SwitchCheck validator = config.approveOutValidator();

        assertNotNull(validator);
        assertFalse(validator.check(null, null, Map.of("action", StsAction.TO_APPROVE.getId())));
    }

    @Test
    void rejectOutValidator_shouldReturnTrue_whenActionIsReject() {
        SwitchCheck validator = config.rejectOutValidator();

        assertNotNull(validator);
        assertTrue(validator.check(null, null, Map.of("action", StsAction.REJECT.getId())));
    }

    @Test
    void rejectOutValidator_shouldReturnFalse_whenActionIsNotReject() {
        SwitchCheck validator = config.rejectOutValidator();

        assertNotNull(validator);
        assertFalse(validator.check(null, null, Map.of("action", StsAction.APPROVE.getId())));
    }
}

```
