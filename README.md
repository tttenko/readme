```java
data class UpdateMetricRequest(

    @field:NotBlank
    @Schema(description = "Название метрики")
    val name: String,

    @field:NotBlank
    @Schema(description = "Единицы измерения")
    val unit: String,

    @field:NotBlank
    @Schema(description = "Направление метрики")
    val direction: String,

    @field:NotEmpty
    @Schema(description = "Режимы работы метрики")
    val agentTypes: Set<String>,

    @field:NotBlank
    @Schema(description = "Описание / формула")
    val description: String,

    @field:NotBlank
    @Schema(description = "Частота сдачи метрики")
    val frequency: String,
)

@PutMapping("/{metricId}")
@Operation(
    summary = "Обновление метрики",
    responses = [ApiResponse(
        responseCode = "200",
        content = [Content(
            mediaType = "application/json",
            schema = Schema(implementation = CreateMetricResponse::class),
        )],
    )],
)
@ExceptionApiResponses
@PreAuthorize(value = "hasAuthority('TRANSFORMATION_OFFICE')")
fun updateMetric(
    @PathVariable metricId: UUID,
    @Valid @RequestBody request: UpdateMetricRequest,
): CreateMetricResponse {
    return metricsService.updateMetric(
        metricId = metricId,
        request = request,
    )
}

private fun MetricsDirectoryEntity.updateFields(
    name: String,
    unit: String,
    direction: String,
    description: String,
    frequency: String,
    agentTypes: Set<String>,
) {
    this.name = name
    this.unit = unit
    this.direction = direction
    this.description = description
    this.frequency = frequency

    this.autonomousApplicability =
        agentTypes.contains(
            MetricAgentType.AUTONOMOUS,
        )

    this.copilotApplicability =
        agentTypes.contains(
            MetricAgentType.COPILOT,
        )

    this.requiresAppealsWork =
        agentTypes.contains(
            MetricAgentType.APPEALS,
        )
}

@Transactional
fun updateMetric(
    metricId: UUID,
    request: UpdateMetricRequest,
): CreateMetricResponse {

    val operationDetails =
        "Обновление метрики"

    val currentUser =
        userInfoProvider.currentUser()

    validateAgentTypes(
        agentTypes = request.agentTypes,
        operationDetails = operationDetails,
    )

    val metric =
        metricsDirectoryRepository.findByIdOrNull(
            id = metricId,
        )
            ?: throwMetricNotFound(
                metricId = metricId,
                operationDetails = operationDetails,
            )

    metric.name = request.name
    metric.unit = request.unit
    metric.direction = request.direction
    metric.description = request.description
    metric.frequency = request.frequency

    metric.autonomousApplicability =
        request.agentTypes.contains(
            MetricAgentType.AUTONOMOUS,
        )

    metric.copilotApplicability =
        request.agentTypes.contains(
            MetricAgentType.COPILOT,
        )

    metric.requiresAppealsWork =
        request.agentTypes.contains(
            MetricAgentType.APPEALS,
        )

    metric.updatedBy = currentUser.id
    metric.updatedAt = dateTimeProvider.currentDateTime()

    val savedMetric =
        metricsDirectoryRepository.save(metric)

    val canBeDeleted =
        initiativeMetricValueRepository
            .countByMetricDirectoryId(
                metricDirectoryId = metricId,
            ) == 0L

    return savedMetric.toCreateMetricResponse(
        canBeDeleted = canBeDeleted,
        lastModifiedBy = currentUser.toFullName(),
    )
}




```
