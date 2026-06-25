```java
@Transactional
    fun update(
        initiativeId: Long,
        request: UpdateInitiativeAgentTypesRequest,
    ): List<InitiativeMetricResponse> {

        val initiative =
            aiAgentRepository.findByIdOrNull(id = initiativeId)
                ?: throw AiBadRequestException(
                    errorCode = INITIATIVE_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[INITIATIVE_NOT_FOUND],
                        initiativeId,
                    ),
                )

        permissionValidator.validate(initiative = initiative)

        val requestedAgentTypes =
            request.agentTypes
                .map { rawAgentType ->
                    validateInitiativeMetricAgentType(
                        agentType = rawAgentType.trim()
                    )
                }
                .toSet()

        updateInitiativeMetricTypes(
            initiative = initiative,
            requestedAgentTypes = requestedAgentTypes
        )

        val metrics =
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = requestedAgentTypes.contains(
                    InitiativeMetricAgentType.AUTONOMOUS
                ),
                copilotSelected = requestedAgentTypes.contains(
                    InitiativeMetricAgentType.COPILOT
                ),
                appealsSelected = requestedAgentTypes.contains(
                    InitiativeMetricAgentType.APPEALS
                ),
            )

        if (metrics.isEmpty()) {
            return emptyList()
        }

        val metricIds =
            metrics.map { metric -> metric.id }
                .toSet()

        val metricValues =
            initiativeMetricValueRepository.findValuesForInitiativeMetrics(
                initiativeId = initiativeId,
                agentTypes = requestedAgentTypes
                    .map { agentType -> agentType.value }
                    .toSet(),
                metricDirectoryIds = metricIds,
                periodFrom = metricResponseBuilder.historyPeriodFrom(),
            )

        return metricResponseBuilder.build(
            metrics = metrics,
            requestedAgentTypes = requestedAgentTypes,
            metricValues = metricValues
        )
    }

    private fun updateInitiativeMetricTypes(
        initiative: AIAgentEntity,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
    ) {
        val initiativeId = initiative.id

        val existingMetricTypes =
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId
            )

        val existingByAgentType =
            existingMetricTypes.associateBy { metricType ->
                metricType.agentType
            }

        val requestedAgentTypeValues =
            requestedAgentTypes
                .map { agentType -> agentType.value }
                .toSet()

        val metricTypesToDelete =
            existingMetricTypes.filter { metricType ->
                metricType.agentType !in requestedAgentTypeValues
            }

        val metricTypeIdsToDelete =
            metricTypesToDelete
                .map { metricType -> metricType.id }
                .toSet()

        if (metricTypeIdsToDelete.isNotEmpty()) {
            initiativeMetricValueRepository.deleteAllByInitiativeMetricTypeIds(
                initiativeMetricTypeIds = metricTypeIdsToDelete
            )

            initiativeMetricTypeRepository.deleteAll(
                entities = metricTypesToDelete
            )
        }

        val metricTypesToCreate =
            requestedAgentTypes
                .filter { agentType ->
                    existingByAgentType[agentType.value] == null
                }
                .map { agentType ->
                    InitiativeMetricTypeEntity(
                        aiAgent = initiative,
                        agentType = agentType.value
                    )
                }

        if (metricTypesToCreate.isNotEmpty()) {
            initiativeMetricTypeRepository.saveAll(
                entities = metricTypesToCreate,
            )
        }
    }

    private fun validateInitiativeMetricAgentType(
        agentType: String
    ): InitiativeMetricAgentType {

        return InitiativeMetricAgentType.fromValue(value = agentType)
            ?: throw AiBadRequestException(
                errorCode = WRONG_INITIATIVE_METRIC_AGENT_TYPE,
                message = MessageFormat.format(
                    messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE],
                    agentType
                )
            )
    }

@Component
class InitiativeMetricResponseBuilder {

    fun historyPeriodFrom(): LocalDate {
        return YearMonth.now()
            .minusMonths(HISTORY_MONTHS_COUNT)
            .atDay(1)
    }

    fun build(
        metrics: List<MetricsDirectoryEntity>,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
        metricValues: List<InitiativeMetricValueEntity>,
    ): List<InitiativeMetricResponse> {

        val metricValuesByMetricAndAgentType =
            metricValues.groupBy { metricValue ->
                MetricValueKey(
                    metricId = metricValue.metricDirectory?.id,
                    agentType = metricValue.initiativeMetricType?.agentType
                )
            }

        return metrics.flatMap { metric ->
            val metricId = metric.id

            val availableAgentTypesForMetric =
                findAvailableAgentTypesForMetric(
                    metric = metric,
                    requestedAgentTypes = requestedAgentTypes
                )

            availableAgentTypesForMetric.map { agentType ->
                val valuesForMetric =
                    metricValuesByMetricAndAgentType[
                        MetricValueKey(
                            metricId = metricId,
                            agentType = agentType.value,
                        )
                    ].orEmpty()

                val latestMetricValue =
                    valuesForMetric.maxByOrNull { metricValue ->
                        metricValue.periodMonth ?: LocalDate.MIN
                    }

                InitiativeMetricResponse(
                    id = metricId,
                    name = metric.name,
                    unit = metric.unit,
                    direction = metric.direction,
                    agentTypes = setOf(agentType.value),
                    isActive = metric.active,
                    description = metric.description,
                    frequency = metric.frequency,
                    metricValue = latestMetricValue?.metricValue,
                    targetValue = latestMetricValue?.targetValue,
                    periods = buildPeriods(
                        metricValues = valuesForMetric,
                    )
                )
            }
        }
    }

    private fun findAvailableAgentTypesForMetric(
        metric: MetricsDirectoryEntity,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
    ): Set<InitiativeMetricAgentType> {

        return requestedAgentTypes
            .filter { agentType ->
                when (agentType) {
                    InitiativeMetricAgentType.AUTONOMOUS ->
                        metric.autonomousApplicability == true

                    InitiativeMetricAgentType.COPILOT ->
                        metric.copilotApplicability == true

                    InitiativeMetricAgentType.APPEALS ->
                        metric.requiresAppealsWork == true
                }
            }
            .toSet()
    }

    private fun buildPeriods(
        metricValues: List<InitiativeMetricValueEntity>
    ): List<InitiativeMetricPeriodResponse> {

        val currentMonth = YearMonth.now()

        val oldestHistoryMonth =
            currentMonth.minusMonths(HISTORY_MONTHS_COUNT)

        val middleHistoryMonth =
            currentMonth.minusMonths(1)

        val monthIndexByMonth =
            mapOf(
                currentMonth to CURRENT_MONTH_INDEX,
                oldestHistoryMonth to OLDEST_HISTORY_MONTH_INDEX,
                middleHistoryMonth to MIDDLE_HISTORY_MONTH_INDEX,
            )

        return metricValues
            .mapNotNull { metricValue ->
                val periodMonth =
                    metricValue.periodMonth ?: return@mapNotNull null

                val yearMonth =
                    YearMonth.from(periodMonth)

                val index =
                    monthIndexByMonth[yearMonth] ?: return@mapNotNull null

                InitiativeMetricPeriodResponse(
                    index = index,
                    period = formatPeriod(
                        periodMonth = periodMonth
                    ),
                    value = metricValue.metricValue,
                )
            }
            .sortedBy { period -> period.index }
    }

    private fun formatPeriod(
        periodMonth: LocalDate
    ): String {
        return YearMonth.from(periodMonth)
            .format(PERIOD_FORMATTER)
    }

    private data class MetricValueKey(
        val metricId: UUID?,
        val agentType: String?,
    )

    companion object {

        private const val HISTORY_MONTHS_COUNT = 2L

        private const val CURRENT_MONTH_INDEX = 0
        private const val OLDEST_HISTORY_MONTH_INDEX = 1
        private const val MIDDLE_HISTORY_MONTH_INDEX = 2

        private val PERIOD_FORMATTER =
            DateTimeFormatter.ofPattern("LLLL yyyy", Locale("ru"))
    }
}

@Component
class InitiativeAgentTypesPermissionValidator(
    private val userInfoProvider: UserInfoProvider,
    private val statusRepository: StatusRepository,
) {

    fun validate(
        initiative: AIAgentEntity
    ) {
        if (!isMvpCompletedOrAfter(initiative = initiative)) {
            return
        }

        if (!currentUserHasRole(role = TRANSFORMATION_OFFICE)) {
            throw AiForbiddenException(
                errorCode = "Only TRANSFORMATION_OFFICE can update agent types from MVP status"
            )
        }
    }

    private fun isMvpCompletedOrAfter(
        initiative: AIAgentEntity
    ): Boolean {

        val currentStatusOrdering =
            initiative.agentStatus?.ordering
                ?: return false

        val mvpCompletedStatus =
            statusRepository.findFirstByCode(
                code = MVP_COMPLETED_STATUS_CODE
            ) ?: return false

        val mvpCompletedOrdering =
            mvpCompletedStatus.ordering
                ?: return false

        return currentStatusOrdering >= mvpCompletedOrdering
    }

    private fun currentUserHasRole(
        role: String
    ): Boolean {
        return userInfoProvider.currentUser().roles.contains(role)
    }

    companion object {

        private const val TRANSFORMATION_OFFICE = "TRANSFORMATION_OFFICE"

        /**
         * status.code для MVP.
         */
        private const val MVP_COMPLETED_STATUS_CODE = "pilot"
    }
}
```
