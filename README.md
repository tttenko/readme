```java
data class UpdateMetricActivityRequest(

    @JsonProperty("isActive")
    @Schema(
        description = "Признак активности метрики",
        example = "true",
        requiredMode = Schema.RequiredMode.REQUIRED,
    )
    val active: Boolean,
)

@PatchMapping("/{metricId}")
@Operation(
    summary = "Изменение активности метрики",
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
open fun updateMetricActivity(
    @PathVariable metricId: UUID,
    @Valid @RequestBody request: UpdateMetricActivityRequest,
): CreateMetricResponse {
    return metricsService.updateMetricActivity(
        metricId = metricId,
        request = request,
    )
}

@Query(
        """
        select count(metricValue.id)
        from InitiativeMetricValueEntity metricValue
        where metricValue.metricDirectory.id = :metricDirectoryId
        """
    )
    fun countByMetricDirectoryId(
        @Param("metricDirectoryId") metricDirectoryId: UUID,
    ): Long

    @Transactional
open fun updateMetricActivity(
    metricId: UUID,
    request: UpdateMetricActivityRequest,
): CreateMetricResponse {
    val operationDetails = "Изменение активности метрики"

    val currentUser = userInfoProvider.currentUser()

    val metric = metricsDirectoryRepository.findByIdOrNull(id = metricId)
        ?: throwMetricNotFound(
            metricId = metricId,
            operationDetails = operationDetails,
        )

    metric.active = request.active
    metric.updatedBy = currentUser.id
    metric.updatedAt = dateTimeProvider.currentDateTime()

    val savedMetric = metricsDirectoryRepository.save(metric)

    val canBeDeleted =
        initiativeMetricValueRepository.countByMetricDirectoryId(
            metricDirectoryId = metricId,
        ) == 0L

    return savedMetric.toCreateMetricResponse(
        canBeDeleted = canBeDeleted,
        lastModifiedBy = currentUser.toFullName(),
    )
}

```
