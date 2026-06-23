```java
@Transactional
fun saveInitiativeMetricValue(
    initiativeId: Long,
    request: SaveInitiativeMetricValueRequest,
): SaveInitiativeMetricValueResponse {

    val agentType = request.agentType?.trim()
    val metricId = request.metricId

    if (agentType.isNullOrBlank()) {
        throw AiBadRequestException(
            errorCode = Metadata.ErrorMessages.REQUIRED_INITIATIVE_METRIC_AGENT_TYPE,
            message = messageProvider[
                Metadata.ErrorMessages.REQUIRED_INITIATIVE_METRIC_AGENT_TYPE
            ],
        )
    }

    if (metricId == null) {
        throw AiBadRequestException(
            errorCode = Metadata.ErrorMessages.REQUIRED_INITIATIVE_METRIC_ID,
            message = messageProvider[
                Metadata.ErrorMessages.REQUIRED_INITIATIVE_METRIC_ID
            ],
        )
    }

    validateInitiativeMetricAgentType(
        agentType = agentType,
    )

    val metricDirectory =
        metricsDirectoryRepository.findByIdOrNull(
            id = metricId,
        ) ?: throw AiBadRequestException(
            errorCode = Metadata.ErrorMessages.INITIATIVE_METRIC_NOT_FOUND,
            message = MessageFormat.format(
                messageProvider[Metadata.ErrorMessages.INITIATIVE_METRIC_NOT_FOUND],
                metricId,
            ),
        )

    val initiativeHasMetricTypes =
        initiativeMetricTypeRepository.existsByAiAgentId(
            initiativeId = initiativeId,
        )

    if (!initiativeHasMetricTypes) {
        throw AiConflictException(
            errorCode = Metadata.ErrorMessages.INITIATIVE_METRIC_TYPES_NOT_FOUND,
            message = MessageFormat.format(
                messageProvider[Metadata.ErrorMessages.INITIATIVE_METRIC_TYPES_NOT_FOUND],
                initiativeId,
            ),
        )
    }

    val initiativeMetricType =
        initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(
            initiativeId = initiativeId,
            agentType = agentType,
        ) ?: throw AiConflictException(
            errorCode = Metadata.ErrorMessages.INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND,
            message = MessageFormat.format(
                messageProvider[Metadata.ErrorMessages.INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND],
                initiativeId,
                agentType,
            ),
        )

    val initiativeMetricTypeId =
        requireNotNull(initiativeMetricType.id) {
            "initiativeMetricType.id must not be null"
        }

    val metricValueEntity =
        initiativeMetricValueRepository.findByInitiativeMetricType_IdAndMetricDirectory_Id(
            initiativeMetricTypeId = initiativeMetricTypeId,
            metricDirectoryId = metricId,
        ) ?: InitiativeMetricValueEntity(
            initiativeMetricType = initiativeMetricType,
            metricDirectory = metricDirectory,
        )

    metricValueEntity.metricValue = request.metricValue
    metricValueEntity.targetValue = request.targetValue

    initiativeMetricValueRepository.save(
        metricValueEntity,
    )

    return SaveInitiativeMetricValueResponse.success()
}
```
