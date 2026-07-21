```java


   @Service
class MetricsAdminService(
    private val metricsDirectoryRepository:
        MetricsDirectoryRepository,
    private val messageProvider: MessageProvider,
    private val userInfoProvider: UserInfoProvider,
    private val dateTimeProvider: DateTimeProvider,
) {

    @Transactional
    fun updatePreAnalyticsSettings(
        metricId: UUID,
        request: UpdateMetricPreAnalyticsRequest,
    ): UpdateMetricPreAnalyticsResponse {
        val metric =
            metricsDirectoryRepository.findByIdOrNull(metricId)
                ?: throw AiNotFoundException(
                    errorCode = METRIC_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[METRIC_NOT_FOUND],
                        metricId,
                    ),
                    operationDetails =
                        "Настройка метрики pre-analytics",
                )

        if (
            request.isPreAnalytics == true &&
            request.code.isNullOrBlank()
        ) {
            throw AiBadRequestException(
                errorCode =
                    PRE_ANALYTICS_CODE_REQUIRED,
                message =
                    "Для метрики pre-analytics необходимо заполнить code",
                operationDetails =
                    "Настройка метрики pre-analytics",
            )
        }

        request.code?.let { code ->
            if (
                metricsDirectoryRepository
                    .existsByCodeAndIdNot(
                        code = code,
                        id = metricId,
                    )
            ) {
                throw AiBadRequestException(
                    errorCode =
                        METRIC_CODE_ALREADY_EXISTS,
                    message =
                        "Метрика с code $code уже существует",
                    operationDetails =
                        "Настройка метрики pre-analytics",
                )
            }
        }

        val currentUser =
            userInfoProvider.currentUser()

        metric.code = request.code
        metric.preAnalytics =
            request.isPreAnalytics
        metric.updatedBy = currentUser.id
        metric.updatedAt =
            dateTimeProvider.currentDateTime()

        val savedMetric =
            metricsDirectoryRepository.save(metric)

        return UpdateMetricPreAnalyticsResponse(
            metricId = requireNotNull(savedMetric.id),
            code = savedMetric.code,
            isPreAnalytics =
                savedMetric.preAnalytics,
        )
    }

    private companion object {
        const val PRE_ANALYTICS_CODE_REQUIRED =
            "metric.pre-analytics.code-required"

        const val METRIC_CODE_ALREADY_EXISTS =
            "metric.code.already-exists"
    }
}
```
