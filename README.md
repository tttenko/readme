```java
data class SaveInitiativeMetricValueRequest(

    @field:NotBlank
    @Schema(
        description = "Режим работы инициативы, для которого сдано значение метрики",
        example = "copilot",
        allowableValues = ["autonomous", "copilot", "appeals"],
    )
    val agentType: String?,

    @field:NotNull
    @JsonProperty("id")
    @Schema(
        description = "Идентификатор метрики, для которой сдано значение",
        example = "550e8400-e29b-41d4-a716-446655440000",
    )
    val metricId: UUID?,

    @Schema(
        description = "Фактическое значение метрики",
        example = "10",
    )
    val metricValue: BigDecimal? = null,

    @Schema(
        description = "Плановое значение метрики",
        example = "20",
    )
    val targetValue: BigDecimal? = null,
)

data class SaveInitiativeMetricValueResponse(
    val code: Int,
    val message: String,
) {
    companion object {

        fun success() =
            SaveInitiativeMetricValueResponse(
                code = 0,
                message = "Metrics saved",
            )
    }
}

@Repository
interface InitiativeMetricTypeRepository :
    JpaRepository<InitiativeMetricTypeEntity, Long> {

    fun existsByAiAgentId(
        initiativeId: Long,
    ): Boolean

    fun findByAiAgentIdAndAgentType(
        initiativeId: Long,
        agentType: String,
    ): InitiativeMetricTypeEntity?
}

fun findByInitiativeMetricTypeIdAndMetricDirectoryId(
        initiativeMetricTypeId: Long,
        metricDirectoryId: UUID,
    ): InitiativeMetricValueEntity?

@PostMapping("/initiatives/{initiativeId}/metrics/value")
@PreAuthorize("hasAnyAuthority('PROJECT_OFFICE', 'CMS_ADMIN', 'TRANSFORMATION_OFFICE')")
@Operation(
    summary = "Сдача метрик по инициативе",
)
fun saveInitiativeMetricValue(
    @PathVariable("initiativeId")
    initiativeId: Long,

    @RequestBody
    @Valid
    request: SaveInitiativeMetricValueRequest,
) = aiAgentService.saveInitiativeMetricValue(
    initiativeId = initiativeId,
    request = request,
)

@Service
class AIAgentService(
    private val aiAgentRepository: AIAgentRepository,
    private val initiativeMetricTypeRepository: InitiativeMetricTypeRepository,
    private val initiativeMetricValueRepository: InitiativeMetricValueRepository,
    private val metricsDirectoryRepository: MetricsDirectoryRepository,
    private val messageProvider: MessageProvider,
) {

    @Transactional
    fun saveInitiativeMetricValue(
        initiativeId: Long,
        request: SaveInitiativeMetricValueRequest,
    ): SaveInitiativeMetricValueResponse {

        val agentType = request.agentType?.trim()
        val metricId = request.metricId

        if (agentType.isNullOrBlank() || metricId == null) {
            throw ResponseStatusException(
                HttpStatus.BAD_REQUEST,
                "agentType and metricId are required",
            )
        }

        validateAgentType(
            agentType = agentType,
        )

        val metricDirectory =
            metricsDirectoryRepository.findByIdOrNull(
                id = metricId,
            ) ?: throw ResponseStatusException(
                HttpStatus.BAD_REQUEST,
                "Metric not found",
            )

        val initiativeHasMetricTypes =
            initiativeMetricTypeRepository.existsByAiAgentId(
                initiativeId = initiativeId,
            )

        if (!initiativeHasMetricTypes) {
            throw ResponseStatusException(
                HttpStatus.CONFLICT,
                "Initiative has no metric agent types",
            )
        }

        val initiativeMetricType =
            initiativeMetricTypeRepository.findByAiAgentIdAndAgentType(
                initiativeId = initiativeId,
                agentType = agentType,
            ) ?: throw ResponseStatusException(
                HttpStatus.CONFLICT,
                "Agent type is not available for initiative",
            )

        val initiativeMetricTypeId =
            requireNotNull(initiativeMetricType.id) {
                "initiativeMetricType.id must not be null"
            }

        val metricValueEntity =
            initiativeMetricValueRepository.findByInitiativeMetricTypeIdAndMetricDirectoryId(
                initiativeMetricTypeId = initiativeMetricTypeId,
                metricDirectoryId = metricId,
            ) ?: InitiativeMetricValueEntity(
                initiativeMetricType = initiativeMetricType,
                metricDirectory = metricDirectory,
            )

        metricValueEntity.metricValue = request.metricValue
        metricValueEntity.targetValue = request.targetValue

        initiativeMetricValueRepository.save(
            metricValueEntity,
        )

        return SaveInitiativeMetricValueResponse.success()
    }

    private fun validateAgentType(
        agentType: String,
    ) {
        val allowedAgentTypes =
            setOf(
                "autonomous",
                "copilot",
                "appeals",
            )

        if (agentType !in allowedAgentTypes) {
            throw ResponseStatusException(
                HttpStatus.BAD_REQUEST,
                "Wrong agentType",
            )
        }
    }
}
```
