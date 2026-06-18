```java
object MetricAgentType {

    const val AUTONOMOUS = "autonomous"
    const val COPILOT = "copilot"
    const val APPEALS = "appeals"

    val ALLOWED_TYPES = setOf(
        AUTONOMOUS,
        COPILOT,
        APPEALS,
    )
}

data class CreateMetricRequest(

    @field:NotBlank
    val name: String,

    @field:NotBlank
    val unit: String,

    @field:NotBlank
    val direction: String,

    val agentTypes: Set<String> = emptySet(),

    @field:NotBlank
    val description: String,

    @field:NotBlank
    val frequency: String,
)

data class CreateMetricResponse(

    val id: UUID,

    val name: String,

    val unit: String,

    val direction: String,

    val agentTypes: Set<String>,

    val isActive: Boolean,

    val description: String,

    val frequency: String,

    val canBeDeleted: Boolean,

    val lastModifiedAt: LocalDateTime,

    val lastModifiedBy: String,
)

interface MetricsDirectoryRepository : JpaRepository<MetricsDirectoryEntity, UUID>

fun MetricsDirectoryEntity.toCreateMetricResponse(
    lastModifiedBy: String,
): CreateMetricResponse {
    return CreateMetricResponse(
        id = id,
        name = name.orEmpty(),
        unit = unit.orEmpty(),
        direction = direction.orEmpty(),
        agentTypes = buildSet {
            if (autonomousApplicability == true) {
                add(MetricAgentType.AUTONOMOUS)
            }

            if (copilotApplicability == true) {
                add(MetricAgentType.COPILOT)
            }

            if (requiresAppealsWork == true) {
                add(MetricAgentType.APPEALS)
            }
        },
        isActive = active ?: false,
        description = description.orEmpty(),
        frequency = frequency.orEmpty(),

        /**
         * Поле canBeDeleted не хранится в БД.
         * При создании метрики всегда возвращается true.
         */
        canBeDeleted = true,

        lastModifiedAt = updatedAt,
        lastModifiedBy = lastModifiedBy,
    )
}

@Validated
@RestController
@RequestMapping("/api/v1/reference/metrics")
@Tag(description = "Metrics", name = "aiqw metrics references")
open class MetricsReferenceController(
    private val metricsReferenceService: MetricsReferenceService,
) {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(
        summary = "Создание метрики",
        responses = [
            ApiResponse(
                responseCode = "201",
                content = [
                    Content(
                        mediaType = "application/json",
                        schema = Schema(implementation = CreateMetricResponse::class),
                    )
                ],
            )
        ],
    )
    @ExceptionApiResponses
    @PreAuthorize("hasAuthority('TRANSFORMATION_OFFICE')")
    open fun createMetric(
        @Valid @RequestBody request: CreateMetricRequest,
    ): CreateMetricResponse {
        return metricsReferenceService.createMetric(
            request = request,
        )
    }
}

@Service
open class MetricsReferenceService(
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val messageProvider: MessageProvider,
    private val userInfoProvider: UserInfoProvider,
    private val authFeignClient: PrmAuthFeignClient,
    private val dateTimeProvider: DateTimeProvider,
) {

    private val log by logger()

    @Transactional
    open fun createMetric(
        request: CreateMetricRequest,
    ): CreateMetricResponse {
        val operationDetails = "Создание метрики"

        validateAgentTypes(
            agentTypes = request.agentTypes,
            operationDetails = operationDetails,
        )

        val userId = userInfoProvider.currentUser().id

        val entity = MetricsDirectoryEntity().apply {
            name = request.name
            unit = request.unit
            direction = request.direction
            description = request.description
            frequency = request.frequency

            /**
             * Корректная логика заполнения флагов:
             *
             * autonomous -> autonomous_applicability
             * copilot -> copilot_applicability
             * appeals -> requires_appeals_work
             */
            autonomousApplicability =
                request.agentTypes.contains(MetricAgentType.AUTONOMOUS)

            copilotApplicability =
                request.agentTypes.contains(MetricAgentType.COPILOT)

            requiresAppealsWork =
                request.agentTypes.contains(MetricAgentType.APPEALS)

            active = true

            /**
             * В БД сохраняем ID пользователя,
             * который создал метрику.
             */
            updatedBy = userId.toString()

            updatedAt = dateTimeProvider.currentDateTime()
        }

        val savedEntity = metricsDirectoryRepository.save(entity)

        val lastModifiedBy =
            try {
                authFeignClient.getUsers(
                    ids = setOf(userId),
                    companyId = null,
                ).body
                    ?.firstOrNull { user ->
                        user.id == userId
                    }
                    ?.fio
                    .orEmpty()
            } catch (e: Exception) {
                log.info(messageProvider[USER_NOT_FOUND])
                ""
            }

        return savedEntity.toCreateMetricResponse(
            lastModifiedBy = lastModifiedBy,
        )
    }

    private fun validateAgentTypes(
        agentTypes: Set<String>,
        operationDetails: String,
    ) {
        val wrongAgentTypes =
            agentTypes - MetricAgentType.ALLOWED_TYPES

        if (wrongAgentTypes.isNotEmpty()) {
            val errorMessage = MessageFormat.format(
                messageProvider[WRONG_METRIC_AGENT_TYPES],
                wrongAgentTypes.joinToString(),
            )

            log.logError(
                operationDetails = operationDetails,
                errorMessage = errorMessage,
            )

            throw AiBadRequestException(
                errorCode = WRONG_METRIC_AGENT_TYPES,
                message = errorMessage,
                operationDetails = operationDetails,
            )
        }
    }

    companion object {

        private const val WRONG_METRIC_AGENT_TYPES = "wrong.metric.agent.types"
        private const val USER_NOT_FOUND = "user.not.found"
    }
}



```
