```java
try {
    initiativeMetricValueRepository.saveAllAndFlush(metricValuesToSave)
} catch (exception: DataIntegrityViolationException) {
    if (exception.isUniqueMetricValueByPeriodViolation()) {
        throw AiBadRequestException(
            errorCode = INITIATIVE_METRIC_VALUE_DUPLICATE,
            message = MessageFormat.format(
                messageProvider[INITIATIVE_METRIC_VALUE_DUPLICATE],
                metricIds.joinToString(),
                agentTypes.joinToString(),
            )
        )
    }

    throw exception
}

return SaveInitiativeMetricValueResponse.success()

private fun DataIntegrityViolationException.isUniqueMetricValueByPeriodViolation(): Boolean {
    val causes = generateSequence(this as Throwable?) { throwable -> throwable.cause }
        .toList()

    return causes
        .filterIsInstance<ConstraintViolationException>()
        .any { constraintViolation ->
            constraintViolation.constraintName == UNIQUE_METRIC_VALUE_BY_PERIOD_CONSTRAINT
        } || causes.any { throwable ->
            throwable.message?.contains(UNIQUE_METRIC_VALUE_BY_PERIOD_CONSTRAINT) == true
        }
}

INITIATIVE_METRIC_VALUE_ALREADY_EXISTS Значение метрики для указанного режима работы за текущий месяц уже существует
```
