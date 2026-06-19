```java
private fun throwMetricNotFound(
    metricId: UUID,
    operationDetails: String,
): Nothing {

    val errorMessage =
        MessageFormat.format(
            messageProvider[METRIC_NOT_FOUND],
            metricId,
        )

    log.logError(
        operationDetails = operationDetails,
        errorMessage = errorMessage,
    )

    throw AiNotFoundException(
        errorCode = METRIC_NOT_FOUND,
        message = errorMessage,
        operationDetails = operationDetails,
    )
}

```
