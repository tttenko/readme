```java


@Service
class InitiativeMetricValueCreator(
    private val messageProvider: MessageProvider,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val initiativeMetricAssignmentRepository: InitiativeMetricAssignmentRepository,
    private val metricApplicabilityRequestRepository: MetricApplicabilityRequestRepository
) {

    @Transactional
    fun saveInitiativeMetricValue(
        initiativeId: Long,
        request: SaveInitiativeMetricValuesRequest
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

        val currentMonth = YearMonth.now()
        val periodMonth = currentMonth.minusMonths(1).atDay(1)
        val submissionMonth = currentMonth.atDay(1)

        val metricIds = request.metricsValues
            .map { metricValueRequest -> metricValueRequest.metricId }
            .toSet()

        val agentTypes = request.metricsValues
            .map { metricValueRequest ->
                validateInitiativeMetricAgentType(agentType = metricValueRequest.agentType.trim()).value
            }
            .toSet()

        val metricDirectoriesMap = metricsDirectoryRepository
            .findAllById(metricIds)
            .associateBy { metricDirectory -> metricDirectory.id }

        validateRequestedMetricsExists(
            requestedMetricIds = metricIds,
            existingMetricIds = metricDirectoriesMap.keys
        )

        val initiativeMetricTypes = initiativeMetricTypeRepository.findAllByAiAgentIdAndAgentTypeIn(
            initiativeId = initiativeId,
            agentTypes = agentTypes
        )

        val initiativeMetricTypesMap = initiativeMetricTypes.associateBy { initiativeMetricType ->
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
            .map { initiativeMetricType -> initiativeMetricType.id }
            .toSet()

        val assignments = initiativeMetricAssignmentRepository.findAllByInitiativeMetricTypeIdsAndMetricIds(
            initiativeMetricTypeIds = initiativeMetricTypeIds,
            metricIds = metricIds
        )

        if (
            hasUnlinkedMetric(
                requests = request.metricsValues,
                initiativeMetricTypesMap = initiativeMetricTypesMap,
                assignments = assignments,
                submissionMonth = submissionMonth
            )
        ) {
            return SaveInitiativeMetricValueResponse.metricUnlink()
        }

        val existingValuesMap = initiativeMetricValueRepository
            .findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                initiativeMetricTypeIds = initiativeMetricTypeIds,
                metricDirectoryIds = metricIds,
                periodMonth = periodMonth
            )
            .associateBy { metricValue ->
                val initiativeMetricTypeId = metricValue.initiativeMetricType?.id
                val metricDirectoryId = metricValue.metricDirectory?.id

                initiativeMetricTypeId to metricDirectoryId
            }

        val metricValuesToSave = request.metricsValues.map { metricValueRequest ->
            val agentType = validateInitiativeMetricAgentType(agentType = metricValueRequest.agentType.trim())
            val metricDirectory = metricDirectoriesMap.getValue(metricValueRequest.metricId)
            val initiativeMetricType = initiativeMetricTypesMap.getValue(agentType.value)
            val initiativeMetricTypeId = initiativeMetricType.id

            val metricValueEntity = existingValuesMap[initiativeMetricTypeId to metricValueRequest.metricId]
                ?: InitiativeMetricValueEntity(
                    initiativeMetricType = initiativeMetricType,
                    metricDirectory = metricDirectory
                )

            metricValueEntity.periodMonth = periodMonth
            metricValueEntity.metricValue = metricValueRequest.metricValue
            metricValueEntity.targetValue = metricValueRequest.targetValue
            metricValueEntity
        }

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
    }

    /**
     * Проверяет, что среди переданных metric + agentType
     * нет метрики, которая была неприменима в месяц сдачи данных.
     *
     * periodMonth — месяц, за который сохраняются значения.
     * submissionMonth — месяц, в котором эти значения фактически сдаются.
     *
     * Отсутствие assignment означает ACTIVE.
     */
    private fun hasUnlinkedMetric(
        requests: List<SaveInitiativeMetricValueRequest>,
        initiativeMetricTypesMap: Map<String, InitiativeMetricTypeEntity>,
        assignments: List<InitiativeMetricAssignmentEntity>,
        submissionMonth: LocalDate,
    ): Boolean {
        if (assignments.isEmpty()) return false

        val assignmentsMap = assignments.associateBy { assignment ->
            assignment.initiativeMetricType.id to assignment.metric.id
        }

        val unlinkedAssignmentIds = metricApplicabilityRequestRepository
            .findUnlinkedAssignmentIdsForSubmissionMonth(
                assignmentIds = assignments.map { it.id }.toSet(),
                submissionMonth = submissionMonth
            )
            .toSet()

        return requests.any { metricValueRequest ->
            val agentType = validateInitiativeMetricAgentType(metricValueRequest.agentType.trim())
            val initiativeMetricType = initiativeMetricTypesMap.getValue(agentType.value)
            val assignment = assignmentsMap[initiativeMetricType.id to metricValueRequest.metricId]

            assignment != null && assignment.id in unlinkedAssignmentIds
        }
    }

    private fun validateInitiativeMetricAgentType(agentType: String): InitiativeMetricAgentType {
        return InitiativeMetricAgentType.fromValue(agentType)
            ?: throw AiBadRequestException(
                errorCode = WRONG_INITIATIVE_METRIC_AGENT_TYPE,
                message = MessageFormat.format(
                    messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE],
                    agentType
                )
            )
    }

    private fun validateRequestedMetricsExists(
        requestedMetricIds: Set<UUID>,
        existingMetricIds: Set<UUID>
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

    private fun validateDuplicates(metricsValues: List<SaveInitiativeMetricValueRequest>) {
        val duplicate = metricsValues
            .groupBy { metricValueRequest ->
                metricValueRequest.agentType.trim() to metricValueRequest.metricId
            }
            .filterValues { groupedValues -> groupedValues.size > 1 }
            .keys
            .firstOrNull()

        if (duplicate != null) {
            throw AiBadRequestException(
                errorCode = INITIATIVE_METRIC_VALUE_ALREADY_EXISTS,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_VALUE_ALREADY_EXISTS],
                    duplicate.second,
                    duplicate.first
                )
            )
        }
    }

    private fun DataIntegrityViolationException.isUniqueMetricValueByPeriodViolation(): Boolean {
        val causes = generateSequence(this as Throwable?) { throwable -> throwable.cause }.toList()

        return causes
            .filterIsInstance<ConstraintViolationException>()
            .any { constraintViolation ->
                constraintViolation.constraintName == INITIATIVE_METRIC_VALUE_ALREADY_EXISTS
            } ||
            causes.any { throwable ->
                throwable.message?.contains(INITIATIVE_METRIC_VALUE_ALREADY_EXISTS) == true
            }
    }
}
```
