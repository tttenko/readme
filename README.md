```java
@GetMapping
    @Operation(
        summary = "Получение справочника метрик",
        responses = [ApiResponse(
            responseCode = "200",
            content = [Content(
                mediaType = "application/json",
                array = ArraySchema(
                    schema = Schema(implementation = CreateMetricResponse::class),
                ),
            )],
        )],
    )
    @ExceptionApiResponses
    @PreAuthorize(value = "hasAuthority('TRANSFORMATION_OFFICE')")
    open fun getMetrics(): List<CreateMetricResponse> {
        return metricsService.getMetrics()
    }

    @Repository
interface InitiativeMetricValueRepository : JpaRepository<InitiativeMetricValueEntity, Long> {

    @Query(
        """
        select distinct metricValue.metricDirectory.id
        from InitiativeMetricValueEntity metricValue
        where metricValue.metricDirectory.id in :metricDirectoryIds
        """
    )
    fun findUsedMetricDirectoryIds(
        @Param("metricDirectoryIds") metricDirectoryIds: Set<UUID>,
    ): Set<UUID>
}

@Transactional(readOnly = true)
    open fun getMetrics(): List<CreateMetricResponse> {
        val metrics = metricsDirectoryRepository.findAll()

        if (metrics.isEmpty()) {
            return emptyList()
        }

        val metricIds = metrics
            .mapNotNull { metric -> metric.id }
            .toSet()

        val usedMetricIds =
            if (metricIds.isEmpty()) {
                emptySet()
            } else {
                initiativeMetricValueRepository.findUsedMetricDirectoryIds(
                    metricDirectoryIds = metricIds,
                )
            }

        val userIds = metrics
            .mapNotNull { metric -> metric.updatedBy }
            .toSet()

        val usersById = getUsersByIds(userIds = userIds)

        return metrics.map { metric ->
            metric.toCreateMetricResponse(
                canBeDeleted = metric.id
                    ?.let { metricId -> metricId !in usedMetricIds }
                    ?: true,
                lastModifiedBy = usersById[metric.updatedBy].orEmpty(),
            )
        }
    }

    private fun getUsersByIds(
        userIds: Set<Long>,
    ): Map<Long, String> {
        if (userIds.isEmpty()) {
            return emptyMap()
        }

        return authFeignClient.getUsers(
            ids = userIds,
            companyId = null,
        )
            .body
            .orEmpty()
            .filter { user -> user.id != null }
            .associate { user ->
                user.id!! to user.toFullName()
            }
    }

    
```
