```java
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
        .associateBy { metricDirectory -> metricDirectory.id }

    val initiativeMetricTypes = initiativeMetricTypeRepository.findAllByAiAgentIdAndAgentTypeIn(
        initiativeId = initiativeId,
        agentTypes = agentTypes,
    )

    val initiativeMetricTypesMap = initiativeMetricTypes
        .associateBy { initiativeMetricType -> initiativeMetricType.agentType }

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
            val initiativeMetricTypeId = requireNotNull(metricValue.initiativeMetricType.id) {
                "initiativeMetricType.id must not be null"
            }

            initiativeMetricTypeId to metricValue.metricDirectory.id
        }

    val metricValuesToSave = request.metricsValues.map { metricValueRequest ->
        val agentType = validateInitiativeMetricAgentType(
            agentType = metricValueRequest.agentType.trim()
        )

        val metricDirectory = metricDirectoriesMap[metricValueRequest.metricId]
            ?: throw AiBadRequestException(
                errorCode = INITIATIVE_METRIC_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_NOT_FOUND],
                    metricValueRequest.metricId
                )
            )

        val initiativeMetricType = initiativeMetricTypesMap[agentType.value]
            ?: throw AiConflictException(
                errorCode = INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND],
                    initiativeId,
                    agentType.value
                )
            )

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

fun findAllByAiAgentIdAndAgentTypeIn(
    initiativeId: Long,
    agentTypes: Set<String>,
): List<InitiativeMetricTypeEntity>

@Query(
    value = """
        select metricValue
        from InitiativeMetricValueEntity metricValue
            join fetch metricValue.initiativeMetricType initiativeMetricType
            join fetch metricValue.metricDirectory metricDirectory
        where initiativeMetricType.id in :initiativeMetricTypeIds
          and metricDirectory.id in :metricDirectoryIds
          and metricValue.periodMonth = :periodMonth
    """
)
fun findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
    @Param("initiativeMetricTypeIds")
    initiativeMetricTypeIds: Set<Long>,

    @Param("metricDirectoryIds")
    metricDirectoryIds: Set<UUID>,

    @Param("periodMonth")
    periodMonth: LocalDate,
): List<InitiativeMetricValueEntity>
```
