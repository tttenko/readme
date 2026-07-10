```java
const val WRONG_INITIATIVE_METRIC_VALUE = "wrong.initiative.metric.value"

wrong.initiative.metric.value=Некорректное значение метрики: {0}

private fun parseMetricValue(value: String?): BigDecimal? {
        val normalizedValue = value?.trim()

        if (normalizedValue.isNullOrEmpty()) {
            return null
        }

        return normalizedValue.toBigDecimalOrNull()
            ?: throw AiBadRequestException(
                errorCode = WRONG_INITIATIVE_METRIC_VALUE,
                message = MessageFormat.format(
                    messageProvider[WRONG_INITIATIVE_METRIC_VALUE],
                    value,
                ),
            )
    }
```
