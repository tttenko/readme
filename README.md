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

        val metricValuesToSave = request.metricsValues.map { metricValueRequest ->
            val agentType = validateInitiativeMetricAgentType(
                agentType = metricValueRequest.agentType.trim()
            )

            val metricDirectory = metricsDirectoryRepository.findByIdOrNull(
                id = metricValueRequest.metricId
            ) ?: throw AiBadRequestException(
                errorCode = INITIATIVE_METRIC_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_NOT_FOUND],
                    metricValueRequest.metricId
                )
            )

            val initiativeMetricType = initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(
                initiativeId = initiativeId,
                agentType = agentType.value
            ) ?: throw AiConflictException(
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

            val metricValueEntity = initiativeMetricValueRepository
                .findByInitiativeMetricTypeIdAndMetricDirectoryIdAndPeriodMonth(
                    initiativeMetricTypeId = initiativeMetricTypeId,
                    metricDirectoryId = metricValueRequest.metricId,
                    periodMonth = periodMonth,
                )
                ?: InitiativeMetricValueEntity(
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
                    messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE]
                )
            )
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

@Query(
        value = """
            select metricValue
            from InitiativeMetricValueEntity metricValue
                join fetch metricValue.metricDirectory metricDirectory
                join fetch metricValue.initiativeMetricType initiativeMetricType
            where initiativeMetricType.id in :initiativeMetricTypeIds
              and metricDirectory.id in :metricDirectoryIds
              and metricValue.periodMonth = :periodMonth
        """
    )
    fun findAllByMetricTypesAndMetricsAndPeriodMonth(
        @Param("initiativeMetricTypeIds")
        initiativeMetricTypeIds: Set<Long>,

        @Param("metricDirectoryIds")
        metricDirectoryIds: Set<UUID>,

        @Param("periodMonth")
        periodMonth: LocalDate,
    ): List<InitiativeMetricValueEntity>

    fun findAllByAiAgentIdAndAgentTypeIn(
        initiativeId: Long,
        agentTypes: Set<String>,
    ): List<InitiativeMetricTypeEntity>

    data class SaveInitiativeMetricValuesRequest(

    @field:NotEmpty
    @field:Valid
    @Schema(description = "Список значений метрик")
    val metricsValues: List<SaveInitiativeMetricValueRequest>
)

const val INITIATIVE_METRIC_VALUE_DUPLICATE = "INITIATIVE_METRIC_VALUE_DUPLICATE"

INITIATIVE_METRIC_VALUE_DUPLICATE=Для метрики {0} и режима работы {1} значение передано несколько раз
```
