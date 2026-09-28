```java

 private fun validateRestoreAvailable(
        request: MetricApplicabilityRequestEntity,
        assignment: InitiativeMetricAssignmentEntity
    ) {
        val approvedRestore =
            request.status == MetricApplicabilityRequestStatus.APPROVED &&
                assignment.applicabilityStatus ==
                    MetricApplicabilityStatus.NOT_APPLICABLE

        val pendingCancel =
            request.status == MetricApplicabilityRequestStatus.PENDING &&
                assignment.applicabilityStatus ==
                    MetricApplicabilityStatus.PENDING &&
                request.isVisibleInOffice

        if (!approvedRestore && !pendingCancel) {
            throw AiBadRequestException(
                errorCode = ACTION_NOT_AVAILABLE,
                message = MessageFormat.format(
                    messageProvider[ACTION_NOT_AVAILABLE],
                    "RESTORE"
                )
            )
        }
    }
```
