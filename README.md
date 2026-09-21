```java
@Query(
    """
        select assignment
        from InitiativeMetricAssignmentEntity assignment
            join fetch assignment.initiativeMetricType initiativeMetricType
            join fetch assignment.metric metric
        where initiativeMetricType.id in :initiativeMetricTypeIds
          and metric.id in :metricIds
    """
)
fun findAllByInitiativeMetricTypeIdsAndMetricIds(
    @Param("initiativeMetricTypeIds")
    initiativeMetricTypeIds: Set<Long>,

    @Param("metricIds")
    metricIds: Set<UUID>,
): List<InitiativeMetricAssignmentEntity>

@Service
class InitiativeMetricValueCreator(
    private val messageProvider: MessageProvider,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val initiativeMetricAssignmentRepository: InitiativeMetricAssignmentRepository,
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

        val periodMonth =
            YearMonth.now()
                .minusMonths(1)
                .atDay(1)

        val metricIds =
            request.metricsValues
                .map { metricValueRequest ->
                    metricValueRequest.metricId
                }
                .toSet()

        val agentTypes =
            request.metricsValues
                .map { metricValueRequest ->
                    validateInitiativeMetricAgentType(
                        agentType = metricValueRequest.agentType.trim()
                    ).value
                }
                .toSet()

        /*
         * Получаем справочные метрики,
         * переданные в запросе.
         */
        val metricDirectoriesMap =
            metricsDirectoryRepository
                .findAllById(metricIds)
                .associateBy { metricDirectory ->
                    requireNotNull(metricDirectory.id) {
                        "metricDirectory.id must not be null"
                    }
                }

        validateRequestedMetricsExists(
            requestedMetricIds = metricIds,
            existingMetricIds = metricDirectoriesMap.keys
        )

        /*
         * Получаем режимы работы инициативы,
         * переданные в запросе.
         */
        val initiativeMetricTypes =
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = initiativeId,
                    agentTypes = agentTypes,
                )

        val initiativeMetricTypesMap =
            initiativeMetricTypes
                .associateBy { initiativeMetricType ->
                    requireNotNull(initiativeMetricType.agentType) {
                        "initiativeMetricType.agentType must not be null"
                    }
                }

        validateRequestedAgentTypesExists(
            initiativeId = initiativeId,
            requestedAgentTypes = agentTypes,
            existingAgentTypes = initiativeMetricTypesMap.keys,
        )

        val initiativeMetricTypeIds =
            initiativeMetricTypes
                .map { initiativeMetricType ->
                    requireNotNull(initiativeMetricType.id) {
                        "initiativeMetricType.id must not be null"
                    }
                }
                .toSet()

        /*
         * Проверяем применимость метрик.
         *
         * assignment отсутствует -> метрика считается ACTIVE.
         * assignment ACTIVE       -> значение можно сохранить.
         *
         * assignment PENDING /
         * NOT_APPLICABLE          -> сохранение запрещено,
         * возвращаем code=2 / Metric unlink.
         */
        val assignments =
            initiativeMetricAssignmentRepository
                .findAllByInitiativeMetricTypeIdsAndMetricIds(
                    initiativeMetricTypeIds = initiativeMetricTypeIds,
                    metricIds = metricIds,
                )

        if (
            hasUnlinkedMetric(
                requests = request.metricsValues,
                initiativeMetricTypesMap = initiativeMetricTypesMap,
                assignments = assignments,
            )
        ) {
            return SaveInitiativeMetricValueResponse.metricUnlink()
        }

        /*
         * Получаем уже существующие значения
         * за текущий отчётный период.
         *
         * Они будут обновлены, а отсутствующие записи —
         * созданы.
         */
        val existingValuesMap =
            initiativeMetricValueRepository
                .findAllByInitiativeMetricTypeIdsAndMetricDirectoryIdsAndPeriodMonth(
                    initiativeMetricTypeIds = initiativeMetricTypeIds,
                    metricDirectoryIds = metricIds,
                    periodMonth = periodMonth,
                )
                .associateBy { metricValue ->

                    val initiativeMetricTypeId =
                        requireNotNull(
                            metricValue.initiativeMetricType?.id
                        ) {
                            "initiativeMetricType.id must not be null"
                        }

                    val metricDirectoryId =
                        requireNotNull(
                            metricValue.metricDirectory?.id
                        ) {
                            "metricDirectory.id must not be null"
                        }

                    initiativeMetricTypeId to metricDirectoryId
                }

        /*
         * Формируем записи для сохранения.
         */
        val metricValuesToSave =
            request.metricsValues.map { metricValueRequest ->

                val agentType =
                    validateInitiativeMetricAgentType(
                        agentType = metricValueRequest.agentType.trim()
                    )

                val metricDirectory =
                    metricDirectoriesMap.getValue(
                        metricValueRequest.metricId
                    )

                val initiativeMetricType =
                    initiativeMetricTypesMap.getValue(
                        agentType.value
                    )

                val initiativeMetricTypeId =
                    requireNotNull(initiativeMetricType.id) {
                        "initiativeMetricType.id must not be null"
                    }

                val metricValueEntity =
                    existingValuesMap[
                        initiativeMetricTypeId to metricValueRequest.metricId
                    ]
                        ?: InitiativeMetricValueEntity(
                            initiativeMetricType = initiativeMetricType,
                            metricDirectory = metricDirectory,
                        )

                metricValueEntity.periodMonth = periodMonth
                metricValueEntity.metricValue =
                    metricValueRequest.metricValue
                metricValueEntity.targetValue =
                    metricValueRequest.targetValue

                metricValueEntity
            }

        try {
            initiativeMetricValueRepository
                .saveAllAndFlush(metricValuesToSave)

        } catch (exception: DataIntegrityViolationException) {

            if (exception.isUniqueMetricValueByPeriodViolation()) {
                throw AiBadRequestException(
                    errorCode = INITIATIVE_METRIC_VALUE_DUPLICATE,
                    message = MessageFormat.format(
                        messageProvider[
                            INITIATIVE_METRIC_VALUE_DUPLICATE
                        ],
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
     * нет метрики, которая сейчас находится в состоянии
     * PENDING или NOT_APPLICABLE.
     *
     * Отсутствие assignment означает ACTIVE.
     */
    private fun hasUnlinkedMetric(
        requests: List<SaveInitiativeMetricValueRequest>,
        initiativeMetricTypesMap: Map<String, InitiativeMetricTypeEntity>,
        assignments: List<InitiativeMetricAssignmentEntity>,
    ): Boolean {

        /*
         * Индексируем assignment по точной бизнес-связке:
         *
         * initiativeMetricTypeId + metricId.
         *
         * Это важно, потому что одна справочная метрика
         * может использоваться сразу несколькими agentType.
         */
        val assignmentsMap =
            assignments.associateBy { assignment ->

                val initiativeMetricTypeId =
                    requireNotNull(
                        assignment.initiativeMetricType.id
                    ) {
                        "initiativeMetricType.id must not be null"
                    }

                val metricId =
                    requireNotNull(
                        assignment.metric.id
                    ) {
                        "metric.id must not be null"
                    }

                initiativeMetricTypeId to metricId
            }

        return requests.any { metricValueRequest ->

            val agentType =
                validateInitiativeMetricAgentType(
                    agentType = metricValueRequest.agentType.trim()
                )

            val initiativeMetricType =
                initiativeMetricTypesMap.getValue(
                    agentType.value
                )

            val initiativeMetricTypeId =
                requireNotNull(initiativeMetricType.id) {
                    "initiativeMetricType.id must not be null"
                }

            val assignment =
                assignmentsMap[
                    initiativeMetricTypeId to metricValueRequest.metricId
                ]

            /*
             * null -> assignment никогда не создавался,
             * поэтому метрика считается ACTIVE.
             */
            assignment != null &&
                assignment.applicabilityStatus != MetricApplicabilityStatus.ACTIVE
        }
    }

    private fun validateInitiativeMetricAgentType(
        agentType: String,
    ): InitiativeMetricAgentType {

        return InitiativeMetricAgentType.fromValue(agentType)
            ?: throw AiBadRequestException(
                errorCode = WRONG_INITIATIVE_METRIC_AGENT_TYPE,
                message = MessageFormat.format(
                    messageProvider[
                        WRONG_INITIATIVE_METRIC_AGENT_TYPE
                    ],
                    agentType,
                )
            )
    }

    private fun validateRequestedMetricsExists(
        requestedMetricIds: Set<UUID>,
        existingMetricIds: Set<UUID>,
    ) {

        val missingMetricIds =
            requestedMetricIds - existingMetricIds

        if (missingMetricIds.isNotEmpty()) {
            throw AiBadRequestException(
                errorCode = INITIATIVE_METRIC_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[
                        INITIATIVE_METRIC_NOT_FOUND
                    ],
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

        val missingAgentTypes =
            requestedAgentTypes - existingAgentTypes

        if (missingAgentTypes.isNotEmpty()) {
            throw AiConflictException(
                errorCode = INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[
                        INITIATIVE_METRIC_AGENT_TYPE_NOT_FOUND
                    ],
                    initiativeId,
                    missingAgentTypes.joinToString()
                )
            )
        }
    }

    private fun validateDuplicates(
        metricsValues: List<SaveInitiativeMetricValueRequest>,
    ) {

        val duplicate =
            metricsValues
                .groupBy { metricValueRequest ->
                    metricValueRequest.agentType.trim() to
                        metricValueRequest.metricId
                }
                .filterValues { groupedValues ->
                    groupedValues.size > 1
                }
                .keys
                .firstOrNull()

        if (duplicate != null) {
            throw AiBadRequestException(
                errorCode = INITIATIVE_METRIC_VALUE_ALREADY_EXISTS,
                message = MessageFormat.format(
                    messageProvider[
                        INITIATIVE_METRIC_VALUE_ALREADY_EXISTS
                    ],
                    duplicate.second,
                    duplicate.first,
                )
            )
        }
    }

    private fun DataIntegrityViolationException
        .isUniqueMetricValueByPeriodViolation(): Boolean {

        val causes =
            generateSequence(
                this as Throwable?
            ) { throwable ->
                throwable.cause
            }.toList()

        return causes
            .filterIsInstance<ConstraintViolationException>()
            .any { constraintViolation ->
                constraintViolation.constraintName ==
                    INITIATIVE_METRIC_VALUE_ALREADY_EXISTS
            } ||
            causes.any { throwable ->
                throwable.message
                    ?.contains(
                        INITIATIVE_METRIC_VALUE_ALREADY_EXISTS
                    ) == true
            }
    }
}



```
