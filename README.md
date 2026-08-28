```java

@Service
class InitiativeAgentTypesUpdater(
    private val messageProvider: MessageProvider,
    private val aiAgentRepository: AIAgentRepository,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val permissionValidator: InitiativeAgentTypesPermissionValidator,
    private val metricResponseBuilder: InitiativeMetricResponseBuilder,
) {

    @Transactional
    fun updateInitiativeAgentType(
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

        permissionValidator.validate(
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
            metricsDirectoryRepository.findApplicableMetrics(
                autonomousSelected =
                    requestedAgentTypes.contains(
                        InitiativeMetricAgentType.AUTONOMOUS,
                    ),
                copilotSelected =
                    requestedAgentTypes.contains(
                        InitiativeMetricAgentType.COPILOT,
                    ),
                appealsSelected =
                    requestedAgentTypes.contains(
                        InitiativeMetricAgentType.APPEALS,
                    ),
            )

        if (metrics.isEmpty()) {
            return emptyList()
        }

        val metricIds =
            metrics
                .map { metric ->
                    metric.id
                }
                .toSet()

        val reportingMonth =
            YearMonth.now()
                .minusMonths(1)

        val previousPeriodMonth =
            reportingMonth.minusMonths(1)

        val metricValues =
            initiativeMetricValueRepository
                .findValuesForInitiativeMetricsInPeriodRange(
                    initiativeId = initiativeId,
                    agentTypes =
                        requestedAgentTypes
                            .map { agentType ->
                                agentType.value
                            }
                            .toSet(),
                    metricDirectoryIds = metricIds,
                    periodFrom = previousPeriodMonth.atDay(1),
                    periodTo = reportingMonth.atDay(1),
                )

        return metricResponseBuilder.build(
            metrics = metrics,
            requestedAgentTypes = requestedAgentTypes,
            metricValues = metricValues,
            reportingMonth = reportingMonth,
        )
    }

    private fun updateInitiativeMetricTypes(
        initiative: AIAgentEntity,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
    ) {
        val initiativeId = initiative.id

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
                .map { metricType ->
                    metricType.id
                }
                .toSet()

        if (metricTypeIdsToDelete.isNotEmpty()) {
            initiativeMetricValueRepository
                .deleteAllByInitiativeMetricTypeIds(
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

    private fun validateInitiativeMetricAgentType(
        agentType: String,
    ): InitiativeMetricAgentType {

        return InitiativeMetricAgentType.fromValue(
            value = agentType,
        )
            ?: throw AiBadRequestException(
                errorCode = WRONG_INITIATIVE_METRIC_AGENT_TYPE,
                message = MessageFormat.format(
                    messageProvider[WRONG_INITIATIVE_METRIC_AGENT_TYPE],
                    agentType,
                ),
            )
    }
}

```
