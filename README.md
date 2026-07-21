```java


    <changeSet
            id="add-pre-analytics-fields-to-metrics-directory"
            author="KoptenkoMV">

        <addColumn tableName="metrics_directory">
            <column
                    name="code"
                    type="VARCHAR(120)"
                    remarks="Стабильный код метрики"/>

            <column
                    name="is_pre_analytics"
                    type="BOOLEAN"
                    remarks="Признак использования метрики в pre-analytics"/>
        </addColumn>

        <addUniqueConstraint
                tableName="metrics_directory"
                columnNames="code"
                constraintName="uk_metrics_directory_code"/>

        <sql>
            ALTER TABLE metrics_directory
                ADD CONSTRAINT chk_metrics_directory_pre_analytics_code
                CHECK (
                    is_pre_analytics IS DISTINCT FROM TRUE
                    OR NULLIF(BTRIM(code), '') IS NOT NULL
                );
        </sql>

        <rollback>
            <sql>
                ALTER TABLE metrics_directory
                    DROP CONSTRAINT IF EXISTS
                        chk_metrics_directory_pre_analytics_code;
            </sql>

            <dropUniqueConstraint
                    tableName="metrics_directory"
                    constraintName="uk_metrics_directory_code"/>

            <dropColumn
                    tableName="metrics_directory"
                    columnName="is_pre_analytics"/>

            <dropColumn
                    tableName="metrics_directory"
                    columnName="code"/>
        </rollback>

    </changeSet>

@Column(
    name = "code",
    length = 120,
)
var code: String? = null

@Column(name = "is_pre_analytics")
var preAnalytics: Boolean? = null

fun findAllByPreAnalyticsTrue():
        List<MetricsDirectoryEntity>

    fun existsByCodeAndIdNot(
        code: String,
        id: UUID,
    ): Boolean

data class UpdateMetricPreAnalyticsRequest(

    @field:Size(max = 120)
    val code: String?,

    @field:NotNull
    @JsonProperty("isPreAnalytics")
    val preAnalytics: Boolean?,
)

data class UpdateMetricPreAnalyticsResponse(
    val metricId: UUID,
    val code: String?,

    @JsonProperty("isPreAnalytics")
    val preAnalytics: Boolean,
)

@Validated
@RestController
@RequestMapping("/api/v1/reference/metrics")
@Tag(
    description = "Metrics pre-analytics administration",
    name = "aiqw metrics pre-analytics administration",
)
class MetricsAdminController(
    private val metricsAdminService: MetricsAdminService,
) {

    @PutMapping("/{metricId}/pre-analytics")
    @Operation(
        summary = "Настройка использования метрики в pre-analytics",
        responses = [
            ApiResponse(
                responseCode = "200",
                content = [
                    Content(
                        mediaType = "application/json",
                        schema = Schema(
                            implementation =
                                UpdateMetricPreAnalyticsResponse::class,
                        ),
                    ),
                ],
            ),
        ],
    )
    @ExceptionApiResponses
    @PreAuthorize("hasAuthority('CMS_ADMIN')")
    fun updatePreAnalyticsSettings(
        @PathVariable
        metricId: UUID,

        @Valid
        @RequestBody
        request: UpdateMetricPreAnalyticsRequest,
    ): UpdateMetricPreAnalyticsResponse {
        return metricsAdminService.updatePreAnalyticsSettings(
            metricId = metricId,
            request = request,
        )
    }
}

@Service
class MetricsAdminService(
    private val metricsDirectoryRepository:
        MetricsDirectoryRepository,
    private val messageProvider: MessageProvider,
    private val userInfoProvider: UserInfoProvider,
    private val dateTimeProvider: DateTimeProvider,
) {

    private val log by logger()

    @Transactional
    fun updatePreAnalyticsSettings(
        metricId: UUID,
        request: UpdateMetricPreAnalyticsRequest,
    ): UpdateMetricPreAnalyticsResponse {
        val operationDetails =
            "Настройка метрики pre-analytics"

        val metric =
            metricsDirectoryRepository.findByIdOrNull(metricId)
                ?: throwMetricNotFound(
                    metricId = metricId,
                    operationDetails = operationDetails,
                )

        val preAnalytics =
            request.preAnalytics
                ?: throwBadRequest(
                    errorCode =
                        PRE_ANALYTICS_FLAG_REQUIRED,
                    errorMessage =
                        messageProvider[
                            PRE_ANALYTICS_FLAG_REQUIRED
                        ],
                    operationDetails = operationDetails,
                )

        val normalizedCode =
            request.code
                ?.trim()
                ?.takeIf(String::isNotEmpty)

        if (preAnalytics && normalizedCode == null) {
            throwBadRequest(
                errorCode =
                    PRE_ANALYTICS_CODE_REQUIRED,
                errorMessage =
                    messageProvider[
                        PRE_ANALYTICS_CODE_REQUIRED
                    ],
                operationDetails = operationDetails,
            )
        }

        if (
            normalizedCode != null &&
            metricsDirectoryRepository
                .existsByCodeAndIdNot(
                    code = normalizedCode,
                    id = metricId,
                )
        ) {
            throwBadRequest(
                errorCode =
                    METRIC_CODE_ALREADY_EXISTS,
                errorMessage =
                    MessageFormat.format(
                        messageProvider[
                            METRIC_CODE_ALREADY_EXISTS
                        ],
                        normalizedCode,
                    ),
                operationDetails = operationDetails,
            )
        }

        val currentUser =
            userInfoProvider.currentUser()

        metric.code = normalizedCode
        metric.preAnalytics = preAnalytics
        metric.updatedBy = currentUser.id
        metric.updatedAt =
            dateTimeProvider.currentDateTime()

        val savedMetric =
            metricsDirectoryRepository.save(metric)

        return UpdateMetricPreAnalyticsResponse(
            metricId = requireNotNull(savedMetric.id),
            code = savedMetric.code,
            preAnalytics =
                savedMetric.preAnalytics == true,
        )
    }

    private fun throwMetricNotFound(
        metricId: UUID,
        operationDetails: String,
    ): Nothing {
        val errorMessage =
            MessageFormat.format(
                messageProvider[METRIC_NOT_FOUND],
                metricId,
            )

        log.logError(
            operationDetails = operationDetails,
            errorMessage = errorMessage,
        )

        throw AiNotFoundException(
            errorCode = METRIC_NOT_FOUND,
            message = errorMessage,
            operationDetails = operationDetails,
        )
    }

    private fun throwBadRequest(
        errorCode: String,
        errorMessage: String,
        operationDetails: String,
    ): Nothing {
        log.logError(
            operationDetails = operationDetails,
            errorMessage = errorMessage,
        )

        throw AiBadRequestException(
            errorCode = errorCode,
            message = errorMessage,
            operationDetails = operationDetails,
        )
    }

    private companion object {
        const val PRE_ANALYTICS_FLAG_REQUIRED =
            "metric.pre-analytics.flag-required"

        const val PRE_ANALYTICS_CODE_REQUIRED =
            "metric.pre-analytics.code-required"

        const val METRIC_CODE_ALREADY_EXISTS =
            "metric.code.already-exists"
    }
}

@Service
class InitiativeMetricPreAnalyticsReader(
    private val messageProvider: MessageProvider,
    private val initiativeMetricTypeRepository:
        InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository:
        InitiativeMetricValueRepository,
    private val metricsDirectoryRepository:
        MetricsDirectoryRepository,
    private val responseBuilder:
        InitiativeMetricPreAnalyticsResponseBuilder,
) {

    @Transactional(readOnly = true)
    fun getPreAnalytics(
        initiativeId: Long,
        now: Instant = Instant.now(),
    ): InitiativeMetricPreAnalyticsResponse {
        val metricTypes =
            initiativeMetricTypeRepository
                .findAllByAiAgentIdAndAgentTypeIn(
                    initiativeId = initiativeId,
                    agentTypes = SUPPORTED_AGENT_TYPES,
                )

        if (metricTypes.isEmpty()) {
            throw AiConflictException(
                errorCode =
                    INITIATIVE_METRIC_TYPES_NOT_FOUND,
                message =
                    MessageFormat.format(
                        messageProvider[
                            INITIATIVE_METRIC_TYPES_NOT_FOUND
                        ],
                        initiativeId,
                    ),
            )
        }

        val metrics =
            metricsDirectoryRepository
                .findAllByPreAnalyticsTrue()

        val metricIds =
            metrics
                .map { metric ->
                    requireNotNull(metric.id)
                }
                .toSet()

        val currentMonth =
            YearMonth.from(
                now.atZone(ZoneOffset.UTC),
            )

        val previousMonth =
            currentMonth.minusMonths(1)

        val beforePreviousMonth =
            currentMonth.minusMonths(2)

        val metricValues =
            if (metricIds.isEmpty()) {
                emptyList()
            } else {
                initiativeMetricValueRepository
                    .findValuesForInitiativeMetrics(
                        initiativeId = initiativeId,
                        agentTypes =
                            SUPPORTED_AGENT_TYPES,
                        metricDirectoryIds = metricIds,
                        periodFrom =
                            beforePreviousMonth.atDay(1),
                    )
            }

        val valuesByKey =
            metricValues
                .filter { metricValue ->
                    metricValue.periodMonth ==
                        previousMonth.atDay(1) ||
                        metricValue.periodMonth ==
                        beforePreviousMonth.atDay(1)
                }
                .associateBy { metricValue ->
                    InitiativeMetricPreAnalyticsResponseBuilder
                        .MetricValueKey(
                            agentType =
                                metricValue
                                    .initiativeMetricType
                                    ?.agentType
                                    .orEmpty(),
                            metricId =
                                requireNotNull(
                                    metricValue
                                        .metricDirectory
                                        ?.id,
                                ),
                            periodMonth =
                                YearMonth.from(
                                    requireNotNull(
                                        metricValue
                                            .periodMonth,
                                    ),
                                ),
                        )
                }

        val metricsAutonomous =
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    AUTONOMOUS.value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = valuesByKey,
                previousMonth = previousMonth,
                beforePreviousMonth =
                    beforePreviousMonth,
            )

        val metricsCopilot =
            responseBuilder.buildMetricsForAgentType(
                agentType =
                    COPILOT.value,
                metricTypes = metricTypes,
                metrics = metrics,
                valuesByKey = valuesByKey,
                previousMonth = previousMonth,
                beforePreviousMonth =
                    beforePreviousMonth,
            )

        val hasSubmittedValues =
            (metricsAutonomous + metricsCopilot)
                .any { metric ->
                    metric.value != null
                }

        return InitiativeMetricPreAnalyticsResponse(
            initiativeId = initiativeId,
            errorCode =
                METRIC_VALUES_NOT_SUBMITTED
                    .takeUnless {
                        hasSubmittedValues
                    },
            periodDisplayText =
                PERIOD_DISPLAY_TEXT_PREFIX +
                    currentMonth
                        .atDay(1)
                        .format(
                            PERIOD_DISPLAY_DATE_FORMATTER,
                        ),
            metricsAutonomous =
                metricsAutonomous,
            metricsCopilot =
                metricsCopilot,
        )
    }

    private companion object {
        const val METRIC_VALUES_NOT_SUBMITTED =
            "metric.not-submitted"

        const val PERIOD_DISPLAY_TEXT_PREFIX =
            "Значения на "

        val PERIOD_DISPLAY_DATE_FORMATTER:
            DateTimeFormatter =
            DateTimeFormatter.ofPattern(
                "dd.MM.yyyy",
            )

        val SUPPORTED_AGENT_TYPES =
            setOf(
                AUTONOMOUS.value,
                COPILOT.value,
            )
    }
}

@Component
class InitiativeMetricPreAnalyticsResponseBuilder {

    fun buildMetricsForAgentType(
        agentType: String,
        metricTypes:
            List<InitiativeMetricTypeEntity>,
        metrics:
            List<MetricsDirectoryEntity>,
        valuesByKey:
            Map<
                MetricValueKey,
                InitiativeMetricValueEntity
            >,
        previousMonth: YearMonth,
        beforePreviousMonth: YearMonth,
    ): List<
        InitiativeMetricPreAnalyticsItemResponse
    > {
        val agentTypeExists =
            metricTypes.any { metricType ->
                metricType.agentType == agentType
            }

        if (!agentTypeExists) {
            return emptyList()
        }

        return metrics.map { metric ->
            val metricId =
                requireNotNull(metric.id)

            val previousValue =
                valuesByKey[
                    MetricValueKey(
                        agentType = agentType,
                        metricId = metricId,
                        periodMonth =
                            previousMonth,
                    )
                ]

            val beforePreviousValue =
                valuesByKey[
                    MetricValueKey(
                        agentType = agentType,
                        metricId = metricId,
                        periodMonth =
                            beforePreviousMonth,
                    )
                ]

            buildMetricResponse(
                metric = metric,
                metricId = metricId,
                previousValue =
                    previousValue,
                beforePreviousValue =
                    beforePreviousValue,
            )
        }
    }

    private fun buildMetricResponse(
        metric: MetricsDirectoryEntity,
        metricId: UUID,
        previousValue:
            InitiativeMetricValueEntity?,
        beforePreviousValue:
            InitiativeMetricValueEntity?,
    ): InitiativeMetricPreAnalyticsItemResponse {
        val metricCode =
            metric.code
                ?.takeIf(String::isNotBlank)
                ?: throw AiInternalServerException(
                    errorCode =
                        PRE_ANALYTICS_CODE_NOT_CONFIGURED,
                    message =
                        "Code is not configured " +
                            "for pre-analytics metric " +
                            metricId,
                )

        val submittedMetricValue =
            previousValue?.metricValue

        if (
            previousValue == null ||
            submittedMetricValue == null
        ) {
            return emptyMetricResponse(
                metric = metric,
                metricId = metricId,
                metricCode = metricCode,
            )
        }

        val comparisonValue =
            beforePreviousValue
                ?.metricValue
                ?.takeIf { value ->
                    submittedMetricValue.compareTo(
                        BigDecimal.ZERO,
                    ) != 0 &&
                        value.compareTo(
                            BigDecimal.ZERO,
                        ) != 0
                }

        val deltaValue =
            comparisonValue?.let { value ->
                val absoluteDeltaValue =
                    submittedMetricValue
                        .subtract(value)
                        .multiply(HUNDRED)
                        .divide(
                            value.abs(),
                            DELTA_SCALE,
                            RoundingMode.HALF_UP,
                        )
                        .abs()

                applyDeltaSign(
                    deltaValue =
                        absoluteDeltaValue,
                    direction =
                        metric.direction,
                    previousValue =
                        submittedMetricValue,
                    beforePreviousValue =
                        value,
                )
            }

        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = metricId,
            code = metricCode,
            name = metric.name,
            unit = metric.unit,
            value = submittedMetricValue,
            targetValue =
                previousValue.targetValue,
            deltaValue = deltaValue,
        )
    }

    private fun emptyMetricResponse(
        metric: MetricsDirectoryEntity,
        metricId: UUID,
        metricCode: String,
    ): InitiativeMetricPreAnalyticsItemResponse {
        return InitiativeMetricPreAnalyticsItemResponse(
            metricId = metricId,
            code = metricCode,
            name = metric.name,
            unit = metric.unit,
            value = null,
            targetValue = null,
            deltaValue = null,
        )
    }

    private fun applyDeltaSign(
        deltaValue: BigDecimal,
        direction: String?,
        previousValue: BigDecimal,
        beforePreviousValue: BigDecimal,
    ): BigDecimal {
        val comparison =
            previousValue.compareTo(
                beforePreviousValue,
            )

        val improved =
            when (direction) {
                MORE_IS_BETTER ->
                    comparison >= 0

                LESS_IS_BETTER ->
                    comparison < 0

                else ->
                    throw AiInternalServerException(
                        errorCode =
                            UNSUPPORTED_METRIC_DIRECTION,
                        message =
                            "Unsupported metric " +
                                "direction: $direction",
                    )
            }

        return if (improved) {
            deltaValue
        } else {
            deltaValue.negate()
        }
    }

    data class MetricValueKey(
        val agentType: String,
        val metricId: UUID,
        val periodMonth: YearMonth,
    )

    private companion object {
        const val MORE_IS_BETTER =
            "more_is_better"

        const val LESS_IS_BETTER =
            "less_is_better"

        const val UNSUPPORTED_METRIC_DIRECTION =
            "unsupported.metric.direction"

        const val PRE_ANALYTICS_CODE_NOT_CONFIGURED =
            "pre-analytics.metric.code-not-configured"

        const val DELTA_SCALE = 0

        val HUNDRED: BigDecimal =
            BigDecimal.valueOf(100)
    }
}
```
