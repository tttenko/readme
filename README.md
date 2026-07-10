```java
@Service
class InitiativeMetricValueCreator(
    private val messageProvider: MessageProvider,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
) {

    @Transactional
    fun saveInitiativeMetricValue(
        initiativeId: Long,
        request: SaveInitiativeMetricValuesRequest,
    ): SaveInitiativeMetricValueResponse {
        validateDuplicates(request.metricsValues)

        if (!initiativeMetricTypeRepository.existsByAiAgentId(initiativeId)) {
            throw AiConflictException(
                errorCode = INITIATIVE_METRIC_TYPES_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_TYPES_NOT_FOUND],
                    initiativeId
                )
            )
        }

        val periodMonth = YearMonth.now().atDay(1)

        val metricIds = request.metricsValues
            .map { metricValueRequest -> metricValueRequest.metricId }
            .toSet()

        val agentTypes = request.metricsValues
            .map { metricValueRequest ->
                validateInitiativeMetricAgentType(
                    agentType = metricValueRequest.agentType.trim()
                ).value
            }
            .toSet()

        val metricDirectoriesMap = metricsDirectoryRepository.findAllById(metricIds)
            .associateBy { metricDirectory ->
                requireNotNull(metricDirectory.id) {
                    "metricDirectory.id must not be null"
                }
            }

        validateRequestedMetricsExists(
            requestedMetricIds = metricIds,
            existingMetricIds = metricDirectoriesMap.keys,
        )

        val initiativeMetricTypes = initiativeMetricTypeRepository.findAllByAiAgentIdAndAgentTypeIn(
            initiativeId = initiativeId,
            agentTypes = agentTypes,
        )

        val initiativeMetricTypesMap = initiativeMetricTypes
            .associateBy { initiativeMetricType ->
                requireNotNull(initiativeMetricType.agentType) {
                    "initiativeMetricType.agentType must not be null"
                }
            }

        validateRequestedAgentTypesExists(
            initiativeId = initiativeId,
            requestedAgentTypes = agentTypes,
            existingAgentTypes = initiativeMetricTypesMap.keys,
        )

        val initiativeMetricTypeIds = initiativeMetricTypes
            .map { initiativeMetricType ->
                requireNotNull(initiativeMetricType.id) {
                    "initiativeMetricType.id must not be null"
                }
            }
            .toSet()

        val existingValuesMap = initiativeMetricValueRepository
            .findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                initiativeMetricTypeIds = initiativeMetricTypeIds,
                metricDirectoryIds = metricIds,
                periodMonth = periodMonth,
            )
            .associateBy { metricValue ->
                val initiativeMetricTypeId = requireNotNull(metricValue.initiativeMetricType?.id) {
                    "initiativeMetricType.id must not be null"
                }

                val metricDirectoryId = requireNotNull(metricValue.metricDirectory?.id) {
                    "metricDirectory.id must not be null"
                }

                initiativeMetricTypeId to metricDirectoryId
            }

        val metricValuesToSave = request.metricsValues.map { metricValueRequest ->
            val agentType = validateInitiativeMetricAgentType(
                agentType = metricValueRequest.agentType.trim()
            )

            val metricDirectory = metricDirectoriesMap.getValue(metricValueRequest.metricId)

            val initiativeMetricType = initiativeMetricTypesMap.getValue(agentType.value)

            val initiativeMetricTypeId = requireNotNull(initiativeMetricType.id) {
                "initiativeMetricType.id must not be null"
            }

            val metricValueEntity = existingValuesMap[
                initiativeMetricTypeId to metricValueRequest.metricId
            ] ?: InitiativeMetricValueEntity(
                initiativeMetricType = initiativeMetricType,
                metricDirectory = metricDirectory,
            )

            metricValueEntity.periodMonth = periodMonth
            metricValueEntity.metricValue = metricValueRequest.metricValue
            metricValueEntity.targetValue = metricValueRequest.targetValue

            metricValueEntity
        }

        initiativeMetricValueRepository.saveAll(metricValuesToSave)

        return SaveInitiativeMetricValueResponse.success()
    }

    private fun validateInitiativeMetricAgentType(
        agentType: String,
    ): InitiativeMetricAgentType {
        return InitiativeMetricAgentType.fromValue(agentType)
            ?: throw AiBadRequestException(
                errorCode = WRONG_INITIATIVE_METRIC_AGENT_TYPE,
                message = MessageFormat.format(
                    messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE],
                    agentType,
                )
            )
    }

    private fun validateRequestedMetricsExists(
        requestedMetricIds: Set<UUID>,
        existingMetricIds: Set<UUID>,
    ) {
        val missingMetricIds = requestedMetricIds - existingMetricIds

        if (missingMetricIds.isNotEmpty()) {
            throw AiBadRequestException(
                errorCode = INITIATIVE_METRIC_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_NOT_FOUND],
                    missingMetricIds.joinToString()
                )
            )
        }
    }

    private fun validateRequestedAgentTypesExists(
        initiativeId: Long,
        requestedAgentTypes: Set<String>,
        existingAgentTypes: Set<String>,
    ) {
        val missingAgentTypes = requestedAgentTypes - existingAgentTypes

        if (missingAgentTypes.isNotEmpty()) {
            throw AiConflictException(
                errorCode = INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND],
                    initiativeId,
                    missingAgentTypes.joinToString()
                )
            )
        }
    }

    private fun validateDuplicates(
        metricsValues: List<SaveInitiativeMetricValueRequest>,
    ) {
        val duplicate = metricsValues
            .groupBy { metricValueRequest ->
                metricValueRequest.agentType.trim() to metricValueRequest.metricId
            }
            .filterValues { groupedValues -> groupedValues.size > 1 }
            .keys
            .firstOrNull()

        if (duplicate != null) {
            throw AiBadRequestException(
                errorCode = INITIATIVE_METRIC_VALUE_DUPLICATE,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_VALUE_DUPLICATE],
                    duplicate.second,
                    duplicate.first,
                )
            )
        }
    }
}
```
