```java
fun buildWithoutValues(
        metrics: List<MetricsDirectoryEntity>,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
    ): List<InitiativeMetricResponse> {

        val requestedAgentTypeValues =
            requestedAgentTypes
                .map { agentType -> agentType.value }
                .toSet()

        return metrics.map { metric ->
            InitiativeMetricResponse(
                id = metric.id,
                name = metric.name,
                unit = metric.unit,
                direction = metric.direction,
                agentTypes = requestedAgentTypeValues,
                isActive = metric.active,
                description = metric.description,
                frequency = metric.frequency,
                metricValue = null,
                targetValue = null,
                periods = emptyList(),
            )
        }
    }


@Service
class InitiativeAgentTypesCreator(
    private val messageProvider: MessageProvider,
    private val aiAgentRepository: AIAgentRepository,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val metricResponseBuilder: InitiativeMetricResponseBuilder,
) {

    @Transactional
    fun create(
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

        val requestedAgentTypes =
            request.agentTypes
                .map { rawAgentType ->
                    validateInitiativeMetricAgentType(
                        agentType = rawAgentType.trim(),
                    )
                }
                .toSet()

        synchronizeAgentTypes(
            initiative = initiative,
            requestedAgentTypes = requestedAgentTypes,
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

        return metricResponseBuilder.buildWithoutValues(
            metrics = metrics,
            requestedAgentTypes = requestedAgentTypes,
        )
    }

    private fun synchronizeAgentTypes(
        initiative: AIAgentEntity,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
    ) {
        val existingAgentTypes =
            initiativeMetricTypeRepository.findAllByAiAgentId(
                initiativeId = initiative.id,
            )

        val existingByAgentType =
            existingAgentTypes.associateBy { it.agentType }

        val requestedAgentTypeValues =
            requestedAgentTypes
                .map { it.value }
                .toSet()

        val agentTypesToDelete =
            existingAgentTypes.filter { existingAgentType ->
                existingAgentType.agentType !in requestedAgentTypeValues
            }

        if (agentTypesToDelete.isNotEmpty()) {
            initiativeMetricTypeRepository.deleteAll(
                entities = agentTypesToDelete,
            )
        }

        val agentTypesToCreate =
            requestedAgentTypes
                .filter { requestedAgentType ->
                    existingByAgentType[requestedAgentType.value] == null
                }
                .map { requestedAgentType ->
                    InitiativeMetricTypeEntity(
                        aiAgent = initiative,
                        agentType = requestedAgentType.value,
                    )
                }

        if (agentTypesToCreate.isNotEmpty()) {
            initiativeMetricTypeRepository.saveAll(
                entities = agentTypesToCreate,
            )
        }
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

@PostMapping("/{initiativeId}/agent-types")
@PreAuthorize("hasAnyAuthority('PROJECT_OFFICE', 'CMS_ADMIN', 'TRANSFORMATION_OFFICE')")
@Operation(summary = "Сохранение режимов работы инициативы")
fun createInitiativeAgentTypes(
    @PathVariable("initiativeId")
    initiativeId: Long,

    @RequestBody
    @Valid
    request: UpdateInitiativeAgentTypesRequest,
) = aiAgentInitiativeService.createInitiativeAgentTypes(
    initiativeId = initiativeId,
    request = request,
)


```
