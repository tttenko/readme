```java
@PutMapping("/{initiativeId}/agent-types")
    @PreAuthorize("hasAnyAuthority('PROJECT_OFFICE', 'CMS_ADMIN', 'TRANSFORMATION_OFFICE')")
    @Operation(summary = "Обновление режимов работы инициативы")
    fun updateInitiativeAgentTypes(
        @PathVariable("initiativeId")
        initiativeId: Long,

        @RequestBody
        @Valid
        request: UpdateInitiativeAgentTypesRequest,
    ) = aiAgentInitiativeService.updateInitiativeAgentTypes(
        initiativeId = initiativeId,
        request = request,
    )

    @Service
class AIAgentInitiativeService(
    private val initiativeMetricValueSaver: InitiativeMetricValueSaver,
    private val initiativeAgentTypesUpdater: InitiativeAgentTypesUpdater,
) {

    fun saveInitiativeMetricValue(
        initiativeId: Long,
        request: SaveInitiativeMetricValueRequest,
    ): SaveInitiativeMetricValueResponse {
        return initiativeMetricValueSaver.save(
            initiativeId = initiativeId,
            request = request,
        )
    }

    fun updateInitiativeAgentTypes(
        initiativeId: Long,
        request: UpdateInitiativeAgentTypesRequest,
    ): List<InitiativeMetricResponse> {
        return initiativeAgentTypesUpdater.update(
            initiativeId = initiativeId,
            request = request,
        )
    }
}

data class UpdateInitiativeAgentTypesRequest(

    @field:NotEmpty
    @Schema(
        description = "Режимы работы инициативы",
        example = "[\"autonomous\", \"copilot\"]",
        allowableValues = ["autonomous", "copilot", "appeals"],
    )
    val agentTypes: Set<String>,
)

data class InitiativeMetricResponse(

    val id: UUID,

    val name: String?,

    val unit: String?,

    val direction: String?,

    val agentTypes: Set<String>,

    @JsonProperty("isActive")
    val isActive: Boolean?,

    val description: String?,

    val frequency: String?,

    val metricValue: BigDecimal?,

    val targetValue: BigDecimal?,

    val periods: List<InitiativeMetricPeriodResponse>,
)

data class InitiativeMetricPeriodResponse(

    val index: Int,

    val period: String,

    val value: BigDecimal?,
)

@Service
class InitiativeMetricValueSaver(
    private val messageProvider: MessageProvider,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
) {

    @Transactional
    fun save(
        initiativeId: Long,
        request: SaveInitiativeMetricValueRequest,
    ): SaveInitiativeMetricValueResponse {

        val agentType =
            validateInitiativeMetricAgentType(
                agentType = request.agentType.trim(),
            )

        val metricId =
            request.metricId

        val periodMonth =
            YearMonth.now()
                .atDay(1)

        val metricDirectory =
            metricsDirectoryRepository.findByIdOrNull(id = metricId)
                ?: throw AiBadRequestException(
                    errorCode = INITIATIVE_METRIC_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[INITIATIVE_METRIC_NOT_FOUND],
                        metricId,
                    ),
                )

        val initiativeHasMetricTypes =
            initiativeMetricTypeRepository.existsByAiAgentId(
                initiativeId = initiativeId,
            )

        if (!initiativeHasMetricTypes) {
            throw AiConflictException(
                errorCode = INITIATIVE_METRIC_TYPES_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_TYPES_NOT_FOUND],
                    initiativeId,
                ),
            )
        }

        val initiativeMetricType =
            initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(
                initiativeId = initiativeId,
                agentType = agentType.value,
            ) ?: throw AiConflictException(
                errorCode = INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND],
                    initiativeId,
                    agentType.value,
                ),
            )

        val initiativeMetricTypeId =
            requireNotNull(initiativeMetricType.id) {
                "initiativeMetricType.id must not be null"
            }

        val metricValueEntity =
            initiativeMetricValueRepository
                .findByInitiativeMetricTypeIdAndMetricDirectoryIdAndPeriodMonth(
                    initiativeMetricTypeId = initiativeMetricTypeId,
                    metricDirectoryId = metricId,
                    periodMonth = periodMonth,
                )
                ?: InitiativeMetricValueEntity(
                    initiativeMetricType = initiativeMetricType,
                    metricDirectory = metricDirectory,
                    periodMonth = periodMonth,
                )

        metricValueEntity.metricValue = request.metricValue
        metricValueEntity.targetValue = request.targetValue

        initiativeMetricValueRepository.save(metricValueEntity)

        return SaveInitiativeMetricValueResponse.success()
    }

    private fun validateInitiativeMetricAgentType(
        agentType: String,
    ): InitiativeMetricAgentType {

        return InitiativeMetricAgentType.fromValue(value = agentType)
            ?: throw AiBadRequestException(
                errorCode = WRONG_INITIATIVE_METRIC_AGENT_TYPE,
                message = MessageFormat.format(
                    messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE],
                    agentType,
                ),
            )
    }
}

@Service
class InitiativeAgentTypesUpdater(
    private val messageProvider: MessageProvider,
    private val userInfoProvider: UserInfoProvider,
    private val aiAgentRepository: AIAgentRepository,
    private val statusRepository: StatusRepository,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
) {

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

        validateUpdateAgentTypesPermission(
            initiative = initiative,
        )

        val requestedAgentTypes =
            request.agentTypes
                .map { rawAgentType ->
                    validateInitiativeMetricAgentType(
                        agentType = rawAgentType.trim(),
                    )
                }
                .toSet()

        updateInitiativeMetricTypes(
            initiative = initiative,
            requestedAgentTypes = requestedAgentTypes,
        )

        val metrics =
            metricsDirectoryRepository.findActiveApplicableMetrics(
                autonomousSelected = requestedAgentTypes.contains(
                    InitiativeMetricAgentType.AUTONOMOUS,
                ),
                copilotSelected = requestedAgentTypes.contains(
                    InitiativeMetricAgentType.COPILOT,
                ),
                appealsSelected = requestedAgentTypes.contains(
                    InitiativeMetricAgentType.APPEALS,
                ),
            )

        if (metrics.isEmpty()) {
            return emptyList()
        }

        val metricIds =
            metrics.mapNotNull { metric ->
                metric.id
            }.toSet()

        val periodFrom =
            YearMonth.now()
                .minusMonths(HISTORY_MONTHS_COUNT)
                .atDay(1)

        val metricValues =
            initiativeMetricValueRepository.findValuesForInitiativeMetrics(
                initiativeId = initiativeId,
                agentTypes = requestedAgentTypes
                    .map { agentType ->
                        agentType.value
                    }
                    .toSet(),
                metricDirectoryIds = metricIds,
                periodFrom = periodFrom,
            )

        return buildMetricResponses(
            metrics = metrics,
            requestedAgentTypes = requestedAgentTypes,
            metricValues = metricValues,
        )
    }

    private fun updateInitiativeMetricTypes(
        initiative: AIAgentEntity,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
    ) {
        val initiativeId =
            requireNotNull(initiative.id) {
                "initiative.id must not be null"
            }

        val existingMetricTypes =
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiativeId,
            )

        val existingByAgentType =
            existingMetricTypes.associateBy { metricType ->
                metricType.agentType
            }

        val requestedAgentTypeValues =
            requestedAgentTypes
                .map { agentType ->
                    agentType.value
                }
                .toSet()

        val metricTypesToDelete =
            existingMetricTypes.filter { metricType ->
                metricType.agentType !in requestedAgentTypeValues
            }

        val metricTypeIdsToDelete =
            metricTypesToDelete
                .mapNotNull { metricType ->
                    metricType.id
                }
                .toSet()

        if (metricTypeIdsToDelete.isNotEmpty()) {
            initiativeMetricValueRepository.deleteAllByInitiativeMetricTypeIds(
                initiativeMetricTypeIds = metricTypeIdsToDelete,
            )

            initiativeMetricTypeRepository.deleteAll(
                metricTypesToDelete,
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
                        agentType = agentType.value,
                    )
                }

        if (metricTypesToCreate.isNotEmpty()) {
            initiativeMetricTypeRepository.saveAll(
                metricTypesToCreate,
            )
        }
    }

    private fun buildMetricResponses(
        metrics: List<MetricsDirectoryEntity>,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
        metricValues: List<InitiativeMetricValueEntity>,
    ): List<InitiativeMetricResponse> {

        val metricValuesByMetricAndAgentType =
            metricValues.groupBy { metricValue ->
                MetricValueKey(
                    metricId = requireNotNull(metricValue.metricDirectory?.id) {
                        "metricValue.metricDirectory.id must not be null"
                    },
                    agentType = requireNotNull(
                        metricValue.initiativeMetricType?.agentType,
                    ) {
                        "metricValue.initiativeMetricType.agentType must not be null"
                    },
                )
            }

        return metrics.flatMap { metric ->
            val metricId =
                requireNotNull(metric.id) {
                    "metric.id must not be null"
                }

            val availableAgentTypesForMetric =
                findAvailableAgentTypesForMetric(
                    metric = metric,
                    requestedAgentTypes = requestedAgentTypes,
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
                    ),
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
        metricValues: List<InitiativeMetricValueEntity>,
    ): List<InitiativeMetricPeriodResponse> {

        val currentMonth =
            YearMonth.now()

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
                    metricValue.periodMonth
                        ?: return@mapNotNull null

                val yearMonth =
                    YearMonth.from(periodMonth)

                val index =
                    monthIndexByMonth[yearMonth]
                        ?: return@mapNotNull null

                InitiativeMetricPeriodResponse(
                    index = index,
                    period = formatPeriod(
                        periodMonth = periodMonth,
                    ),
                    value = metricValue.metricValue,
                )
            }
            .sortedBy { period ->
                period.index
            }
    }

    private fun formatPeriod(
        periodMonth: LocalDate,
    ): String {
        return YearMonth.from(periodMonth)
            .format(PERIOD_FORMATTER)
    }

    private fun validateUpdateAgentTypesPermission(
        initiative: AIAgentEntity,
    ) {
        if (!isMvpCompletedOrAfter(initiative = initiative)) {
            return
        }

        if (!currentUserHasRole(TRANSFORMATION_OFFICE)) {
            throw AccessDeniedException(
                "Only TRANSFORMATION_OFFICE can update agent types from MVP status",
            )
        }
    }

    private fun isMvpCompletedOrAfter(
        initiative: AIAgentEntity,
    ): Boolean {
        val currentStatusOrdering =
            initiative.agentStatus?.ordering
                ?: return false

        val mvpCompletedStatus =
            statusRepository.findFirstByCode(
                code = MVP_COMPLETED_STATUS_CODE,
            ) ?: return false

        val mvpCompletedOrdering =
            mvpCompletedStatus.ordering
                ?: return false

        return currentStatusOrdering >= mvpCompletedOrdering
    }

    private fun currentUserHasRole(
        role: String,
    ): Boolean {
        return userInfoProvider.roles
            .contains(role)
    }

    private fun validateInitiativeMetricAgentType(
        agentType: String,
    ): InitiativeMetricAgentType {

        return InitiativeMetricAgentType.fromValue(value = agentType)
            ?: throw AiBadRequestException(
                errorCode = WRONG_INITIATIVE_METRIC_AGENT_TYPE,
                message = MessageFormat.format(
                    messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE],
                    agentType,
                ),
            )
    }

    private data class MetricValueKey(
        val metricId: UUID,
        val agentType: String,
    )

    companion object {

        private const val TRANSFORMATION_OFFICE = "TRANSFORMATION_OFFICE"

        /**
         * Статус MVP из таблицы status:
         * name = MVP,
         * code = pilot,
         * ordering = 40.
         */
        private const val MVP_COMPLETED_STATUS_CODE = "pilot"

        /**
         * Текущий месяц + два предыдущих.
         */
        private const val HISTORY_MONTHS_COUNT = 2L

        private const val CURRENT_MONTH_INDEX = 0

        private const val OLDEST_HISTORY_MONTH_INDEX = 1

        private const val MIDDLE_HISTORY_MONTH_INDEX = 2

        private val PERIOD_FORMATTER =
            DateTimeFormatter.ofPattern(
                "LLLL yyyy",
                Locale("ru"),
            )
    }
}

TYPE
fun findAllByAiAgentId(
        initiativeId: Long,
    ): List<InitiativeMetricTypeEntity>


VALUE
fun findByInitiativeMetricTypeIdAndMetricDirectoryIdAndPeriodMonth(
        initiativeMetricTypeId: Long,
        metricDirectoryId: UUID,
        periodMonth: LocalDate,
    ): InitiativeMetricValueEntity?

    @Query(
        value = """
            select metricValue
            from InitiativeMetricValueEntity metricValue
                join fetch metricValue.metricDirectory metricDirectory
                join fetch metricValue.initiativeMetricType initiativeMetricType
            where initiativeMetricType.aiAgent.id = :initiativeId
                and initiativeMetricType.agentType in :agentTypes
                and metricDirectory.id in :metricDirectoryIds
                and metricValue.periodMonth >= :periodFrom
        """
    )
    fun findValuesForInitiativeMetrics(
        @Param("initiativeId")
        initiativeId: Long,

        @Param("agentTypes")
        agentTypes: Set<String>,

        @Param("metricDirectoryIds")
        metricDirectoryIds: Set<UUID>,

        @Param("periodFrom")
        periodFrom: LocalDate,
    ): List<InitiativeMetricValueEntity>

    @Modifying
    @Query(
        value = """
            delete from InitiativeMetricValueEntity metricValue
            where metricValue.initiativeMetricType.id in :initiativeMetricTypeIds
        """
    )
    fun deleteAllByInitiativeMetricTypeIds(
        @Param("initiativeMetricTypeIds")
        initiativeMetricTypeIds: Set<Long>,
    )

METRICS
@Query(
        value = """
            select distinct metric
            from MetricsDirectoryEntity metric
            where metric.active = true
                and (
                    (:autonomousSelected = true and metric.autonomousApplicability = true)
                    or (:copilotSelected = true and metric.copilotApplicability = true)
                    or (:appealsSelected = true and metric.requiresAppealsWork = true)
                )
        """
    )
    fun findActiveApplicableMetrics(
        @Param("autonomousSelected")
        autonomousSelected: Boolean,

        @Param("copilotSelected")
        copilotSelected: Boolean,

        @Param("appealsSelected")
        appealsSelected: Boolean,
    ): List<MetricsDirectoryEntity>

<changeSet id="add-unique-initiative-metric-value-by-period" author="your_name">
    <addUniqueConstraint
        tableName="initiative_metric_value"
        columnNames="initiative_agent_type_id, metric_directory_id, period_month"
        constraintName="uq_initiative_metric_value_type_metric_period"/>
</changeSet>
```
