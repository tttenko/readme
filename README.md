```java
private fun prepareStatuses() {
    analysisStatus = requireNotNull(
        statusRepository.findFirstByCode(ANALYSIS_STATUS_CODE)
    ) {
        "Status '$ANALYSIS_STATUS_CODE' must exist in integration database"
    }

    developmentStatus = requireNotNull(
        statusRepository.findFirstByCode(DEVELOPMENT_STATUS_CODE)
    ) {
        "Status '$DEVELOPMENT_STATUS_CODE' must exist in integration database"
    }

    requireNotNull(
        statusRepository.findFirstByCode(TARGET_SOLUTION_STATUS_CODE)
    ) {
        "Status '$TARGET_SOLUTION_STATUS_CODE' must exist in integration database"
    }
}

```
