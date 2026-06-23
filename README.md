```java
open class AiConflictException(
    errorCode: String,
    message: String? = null,
    fieldErrors: List<FieldError>? = null,
    operationDetails: String? = "",
    formErrors: List<String>? = null,
) : AiResponseException(
    operationDetails = operationDetails,
    status = HttpStatus.CONFLICT,
    errorCode = errorCode,
    message = message,
    fieldErrors = fieldErrors,
    formErrors = formErrors,
)

const val REQUIRED_INITIATIVE_METRIC_AGENT_TYPE =
            "required.initiative.metric.agent.type"

        const val REQUIRED_INITIATIVE_METRIC_ID =
            "required.initiative.metric.id"

        const val WRONG_INITIATIVE_METRIC_AGENT_TYPE =
            "wrong.initiative.metric.agent.type"

        const val INITIATIVE_METRIC_NOT_FOUND =
            "initiative.metric.not.found"

        const val INITIATIVE_METRIC_TYPES_NOT_FOUND =
            "initiative.metric.types.not.found"

        const val INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND =
            "initiative.metric.agent.type.not.found"

required.initiative.metric.agent.type=Не передан обязательный параметр agentType
required.initiative.metric.id=Не передан обязательный параметр id метрики
wrong.initiative.metric.agent.type=Недопустимый режим работы инициативы: {0}
initiative.metric.not.found=Метрика с идентификатором {0} не найдена
initiative.metric.types.not.found=Для инициативы с идентификатором {0} не найдены режимы работы
initiative.metric.agent.type.not.found=Для инициативы с идентификатором {0} не найден режим работы: {1}

required.initiative.metric.agent.type=Required parameter agentType is missing
required.initiative.metric.id=Required metric id parameter is missing
wrong.initiative.metric.agent.type=Unsupported initiative agent type: {0}
initiative.metric.not.found=Metric with id {0} was not found
initiative.metric.types.not.found=No metric agent types were found for initiative with id {0}
initiative.metric.agent.type.not.found=Agent type {1} was not found for initiative with id {0}
```
