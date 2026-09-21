```java
 val applicabilityStatus: MetricApplicabilityStatus = MetricApplicabilityStatus.ACTIVE,
    val resumePeriod: LocalDate? = null,

    /**
     * Возвращает все существующие assignment для инициативы
     * и набора метрик.
     *
     * Отсутствие assignment для конкретной пары metric + agentType
     * означает applicabilityStatus=ACTIVE.
     */
    @Query(
        """
        select assignment
        from InitiativeMetricAssignmentEntity assignment
            join fetch assignment.initiativeMetricType initiativeMetricType
            join fetch assignment.metric metric
        where initiativeMetricType.aiAgent.id = :initiativeId
          and metric.id in :metricIds
        """
    )
    fun findAllByInitiativeIdAndMetricIds(
        @Param("initiativeId") initiativeId: Long,
        @Param("metricIds") metricIds: Set<UUID>,
    ): List<InitiativeMetricAssignmentEntity>


@Query(
    """
    select request
    from MetricApplicabilityRequestEntity request
    where request.initiativeMetricAssignment.id in :assignmentIds
      and not exists (
          select newerRequest.id
          from MetricApplicabilityRequestEntity newerRequest
          where newerRequest.initiativeMetricAssignment.id =
                    request.initiativeMetricAssignment.id
            and (
                newerRequest.createdAt > request.createdAt
                or (
                    newerRequest.createdAt = request.createdAt
                    and newerRequest.id > request.id
                )
            )
      )
    """
)
fun findLatestByAssignmentIds(
    @Param("assignmentIds") assignmentIds: Set<Long>
): List<MetricApplicabilityRequestEntity>

private val initiativeMetricAssignmentRepository: InitiativeMetricAssignmentRepository,
    private val metricApplicabilityRequestRepository: MetricApplicabilityRequestRepository,

@Transactional(readOnly = true)
fun getInitiativeMetricValues(
    initiativeId: Long
): List<InitiativeMetricResponse> {

    val metricTypes =
        initiativeMetricTypeRepository.findAllByAiAgentId(
            initiativeId = initiativeId
        )

    if (metricTypes.isEmpty()) {
        return emptyList()
    }

    val requestedAgentTypes =
        metricTypes
            .map { metricType ->
                InitiativeMetricAgentType
                    .fromValue(metricType.agentType.orEmpty())
                    ?: throw AiBadRequestException(
                        errorCode = WRONG_INITIATIVE_METRIC_AGENT_TYPE,
                        message = MessageFormat.format(
                            messageProvider[
                                WRONG_INITIATIVE_METRIC_AGENT_TYPE
                            ],
                            metricType.agentType,
                        ),
                    )
            }
            .toSet()

    val metrics =
        metricsDirectoryRepository.findApplicableMetrics(
            autonomousSelected =
                requestedAgentTypes.contains(
                    InitiativeMetricAgentType.AUTONOMOUS
                ),
            copilotSelected =
                requestedAgentTypes.contains(
                    InitiativeMetricAgentType.COPILOT
                ),
            appealsSelected =
                requestedAgentTypes.contains(
                    InitiativeMetricAgentType.APPEALS
                )
        )

    if (metrics.isEmpty()) {
        return emptyList()
    }

    val metricIds = metrics
        .map { metric -> metric.id }
        .toSet()

    /*
     * Получаем состояния применимости одним запросом.
     *
     * В БД assignment существует только для тех metric + agentType,
     * по которым уже запускался процесс неприменимости.
     *
     * Если assignment отсутствует, ниже считаем метрику ACTIVE.
     */
    val assignments =
        initiativeMetricAssignmentRepository
            .findAllByInitiativeIdAndMetricIds(
                initiativeId = initiativeId,
                metricIds = metricIds,
            )

    /*
     * Для каждого assignment нужна актуальная заявка,
     * чтобы получить resumePeriod.
     *
     * Актуальность:
     * createdAt DESC, id DESC.
     */
    val assignmentIds =
        assignments
            .map { assignment -> assignment.id }
            .toSet()

    val latestRequestsByAssignmentId =
        if (assignmentIds.isEmpty()) {
            emptyMap()
        } else {
            metricApplicabilityRequestRepository
                .findLatestByAssignmentIds(assignmentIds)
                .associateBy { request ->
                    request.initiativeMetricAssignment.id
                }
        }

    /*
     * Индексируем состояние именно по:
     *
     * metricId + agentType
     *
     * потому что одна справочная метрика может одновременно
     * существовать для autonomous и copilot с разными статусами.
     */
    val applicabilityByMetricAndAgentType =
        assignments.associate { assignment ->

            val key =
                MetricApplicabilityKey(
                    metricId = assignment.metric.id,
                    agentType =
                        assignment
                            .initiativeMetricType
                            .agentType
                            .orEmpty(),
                )

            val latestRequest =
                latestRequestsByAssignmentId[assignment.id]

            key to MetricApplicabilityData(
                status = assignment.applicabilityStatus,
                resumePeriod = latestRequest?.resumePeriod,
            )
        }

    val reportingMonth = YearMonth.now().minusMonths(1)
    val previousPeriodMonth = reportingMonth.minusMonths(1)

    val hasReportingMonthValues =
        initiativeMetricValueRepository
            .existsByInitiativeMetricTypeAiAgentIdAndPeriodMonth(
                initiativeId = initiativeId,
                periodMonth = reportingMonth.atDay(1),
            )

    val metricValues =
        initiativeMetricValueRepository
            .findValuesForInitiativeMetricsInPeriodRange(
                initiativeId = initiativeId,
                agentTypes =
                    requestedAgentTypes
                        .map { agentType -> agentType.value }
                        .toSet(),
                metricDirectoryIds = metricIds,
                periodFrom = previousPeriodMonth.atDay(1),
                periodTo = reportingMonth.atDay(1),
            )

    val metricIdsWithSubmittedValue =
        metricValues
            .asSequence()
            .filter { metricValue ->
                metricValue.metricValue != null ||
                    metricValue.targetValue != null
            }
            .mapNotNull { metricValue ->
                metricValue.metricDirectory?.id
            }
            .toSet()

    return metricResponseBuilder
        .build(
            metrics = metrics,
            requestedAgentTypes = requestedAgentTypes,
            metricValues = metricValues,
            reportingMonth = reportingMonth,
            clearRegularMetricValue = !hasReportingMonthValues,
        )
        /*
         * Обогащаем уже построенный response состоянием применимости.
         */
        .map { response ->

            val applicability =
                applicabilityByMetricAndAgentType[
                    MetricApplicabilityKey(
                        metricId = response.id,
                        agentType = response.agentType,
                    )
                ]

            response.copy(
                applicabilityStatus =
                    applicability?.status
                        ?: MetricApplicabilityStatus.ACTIVE,
                resumePeriod =
                    applicability?.resumePeriod,
            )
        }
        .filter { response ->
            response.isActive != false ||
                response.id in metricIdsWithSubmittedValue
        }
        .sortedBy { response ->
            response.isActive == false
        }
}

private data class MetricApplicabilityKey(
    val metricId: UUID,
    val agentType: String,
)

private data class MetricApplicabilityData(
    val status: MetricApplicabilityStatus,
    val resumePeriod: LocalDate?,
)


```
