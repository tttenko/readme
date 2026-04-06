```java
class StsTrackerValidatorsConfigTest {

    private static final String ACTION_CTX_KEY = "action";

    private final StsTrackerValidatorsConfig config = new StsTrackerValidatorsConfig();

    @Test
    void toApproveInValidator_shouldReturnTrue_whenActionIsToApprove() {
        assertValidator(config.toApproveInValidator(), StsAction.TO_APPROVE, true);
    }

    @Test
    void toApproveInValidator_shouldReturnFalse_whenActionIsNotToApprove() {
        assertValidator(config.toApproveInValidator(), StsAction.APPROVE, false);
    }

    @Test
    void approveInValidator_shouldReturnTrue_whenActionIsApprove() {
        assertValidator(config.approveInValidator(), StsAction.APPROVE, true);
    }

    @Test
    void approveInValidator_shouldReturnFalse_whenActionIsNotApprove() {
        assertValidator(config.approveInValidator(), StsAction.REJECT, false);
    }

    @Test
    void rejectInValidator_shouldReturnTrue_whenActionIsReject() {
        assertValidator(config.rejectInValidator(), StsAction.REJECT, true);
    }

    @Test
    void rejectInValidator_shouldReturnFalse_whenActionIsNotReject() {
        assertValidator(config.rejectInValidator(), StsAction.APPROVE, false);
    }

    @Test
    void toApproveOutValidator_shouldReturnTrue_whenActionIsToApprove() {
        assertValidator(config.toApproveOutValidator(), StsAction.TO_APPROVE, true);
    }

    @Test
    void toApproveOutValidator_shouldReturnFalse_whenActionIsNotToApprove() {
        assertValidator(config.toApproveOutValidator(), StsAction.REJECT, false);
    }

    @Test
    void approveOutValidator_shouldReturnTrue_whenActionIsApprove() {
        assertValidator(config.approveOutValidator(), StsAction.APPROVE, true);
    }

    @Test
    void approveOutValidator_shouldReturnFalse_whenActionIsNotApprove() {
        assertValidator(config.approveOutValidator(), StsAction.TO_APPROVE, false);
    }

    @Test
    void rejectOutValidator_shouldReturnTrue_whenActionIsReject() {
        assertValidator(config.rejectOutValidator(), StsAction.REJECT, true);
    }

    @Test
    void rejectOutValidator_shouldReturnFalse_whenActionIsNotReject() {
        assertValidator(config.rejectOutValidator(), StsAction.APPROVE, false);
    }

    private void assertValidator(SwitchCheck validator, StsAction action, boolean expected) {
        assertNotNull(validator);

        boolean actual = validator.check(
                null,
                null,
                Map.of(ACTION_CTX_KEY, action.getId())
        );

        if (expected) {
            assertTrue(actual);
        } else {
            assertFalse(actual);
        }
    }
}
```
