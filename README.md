```java
@ConfigurationProperties(prefix = "meta-ai-objects.deviations")
data class MetaAiObjectsDeviationProperties(
    val enabled: Boolean = true,
    val rules: Map<MetaAiObjectDeviationCode, Rule> = emptyMap()
) {

    data class Rule(
        val enabled: Boolean = true,
        val priority: Int,
        val title: String,
        val description: String
    )
}

meta-ai-objects:
  deviations:
    enabled: true
    rules:
      AI_OBJECT_NAME_NOT_FILLED:
        enabled: true
        priority: 20
        title: "Не заполнено наименование"
        description: "Для агента или LLM-решения не указано наименование"

      AI_OBJECT_TYPE_NOT_FILLED:
        enabled: true
        priority: 30
        title: "Не указан тип"
        description: "Не указано, является объект агентом или LLM-решением"

AI_OBJECT_REQUIRED:
  enabled: true
  priority: 70
  title: "Добавить LLM-решение или ИИ-агента"
  description: "Не добавлен LLM-решение или ИИ-агент"
  cta: "Перейти"

enum class MetaAiObjectDeviationCode {
    AI_OBJECT_NAME_NOT_FILLED,
    AI_OBJECT_TYPE_NOT_FILLED
}


import io.swagger.v3.oas.annotations.Operation
import org.springframework.http.MediaType
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

/**
 * API связанных AI-объектов инициатив.
 */
@RestController
@RequestMapping("/api/ai/v1/ai-agent")
class MetaAiObjectsController(
    private val metaAiObjectsService: MetaAiObjectsService
) {

    /**
     * Возвращает активные AI-объекты инициативы
     * с рассчитанными отклонениями.
     */
    @GetMapping(
        "/{initiativeId}/meta-ai-objects",
        produces = [MediaType.APPLICATION_JSON_VALUE]
    )
    @PreAuthorize("hasAnyAuthority('TRANSFORMATION_OFFICE')")
    @Operation(summary = "Получение данных по связанным ИИ-агентам и LLM-решениям")
    fun getMetaAiObjects(
        @PathVariable initiativeId: Long
    ): MetaAiObjectsResponse {
        return metaAiObjectsService.getMetaAiObjects(initiativeId)
    }
}

/**
 * Получение связанных AI-объектов инициативы.
 */
@Service
class MetaAiObjectsService(
    private val aiAgentRepository: AIAgentRepository,
    private val jiraIssueRepository: JiraIssueRepository,
    private val metaAiObjectRepository: InitiativeMetaAiObjectRepository,
    private val deviationCalculator: MetaAiObjectDeviationCalculator,
    private val messageProvider: MessageProvider
) {

    /**
     * Возвращает активные AI-объекты инициативы с рассчитанными отклонениями.
     *
     * @param initiativeId идентификатор инициативы
     * @return список активных объектов, включая объекты без дополнительных сведений
     */
    @Transactional(readOnly = true)
    fun getMetaAiObjects(initiativeId: Long): MetaAiObjectsResponse {
        if (initiativeId <= 0) {
            throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.IncorrectInputValue,
                message = messageProvider[Metadata.ErrorMessages.IncorrectInputValue]
            )
        }

        aiAgentRepository.findByIdOrNull(initiativeId)
            ?.takeIf { it.disabled != true }
            ?: throw AiNotFoundException(
                errorCode = Metadata.ErrorMessages.INITIATIVE_NOT_FOUND,
                message = messageProvider[Metadata.ErrorMessages.INITIATIVE_NOT_FOUND]
            )

        val crossgoal = jiraIssueRepository.findCrossgoalKeysByInitiativeId(initiativeId)
            .firstOrNull()
            ?: return MetaAiObjectsResponse(emptyList())

        val objects = metaAiObjectRepository.findActiveByInitiativeIdAndCrossgoal(
            initiativeId = initiativeId,
            crossgoal = crossgoal
        )

        return MetaAiObjectsResponse(
            objects = objects.map { obj ->
                val deviations = deviationCalculator.calculate(obj)

                MetaAiObjectResponse(
                    id = obj.id,
                    ci = obj.ciSolution,
                    name = obj.name,
                    type = when (obj.isAgent) {
                        true -> MetaAiObjectType.AGENT
                        false -> MetaAiObjectType.LLM_SOLUTION
                        null -> null
                    },
                    gigaUsageId = obj.gigausage,
                    metaLink = obj.metaLink,
                    deviationCount = deviations.size,
                    deviations = deviations
                )
            }
        )
    }
}

/**
 * Возвращает активные AI-объекты указанной инициативы по её CROSSGOAL.
 *
 * В выборку входят объекты с незаполненными дополнительными сведениями.
 */
@Query("""
    SELECT obj
    FROM InitiativeMetaAiObjectEntity obj
    WHERE obj.initiative.id = :initiativeId
      AND obj.crossgoal = :crossgoal
      AND obj.isActive = true
    ORDER BY obj.id ASC
""")
fun findActiveByInitiativeIdAndCrossgoal(
    @Param("initiativeId") initiativeId: Long,
    @Param("crossgoal") crossgoal: String
): List<InitiativeMetaAiObjectEntity>

/**
 * Возвращает CROSSGOAL-ключи инициативы.
 *
 * Используются записи Jira проекта CROSSGOAL с типом initiative.
 */
@Query("""
    SELECT j.jiraKey
    FROM JiraIssueEntity j
    WHERE j.agent.id = :initiativeId
      AND LOWER(j.project) = 'crossgoal'
      AND j.type = 'initiative'
      AND j.jiraKey IS NOT NULL
      AND TRIM(j.jiraKey) <> ''
    ORDER BY j.id ASC
""")
fun findCrossgoalKeysByInitiativeId(
    @Param("initiativeId") initiativeId: Long
): List<String>

/**
 * Расчёт отклонений связанных AI-объектов.
 *
 * Условия формирования определяются кодами правил.
 * Активность, приоритет и тексты отклонений берутся из конфигурации.
 */
@Service
class MetaAiObjectDeviationCalculator(
    private val properties: MetaAiObjectsDeviationProperties
) {

    /**
     * Рассчитывает активные отклонения объекта и сортирует их по приоритету.
     */
    fun calculate(obj: InitiativeMetaAiObjectEntity): List<MetaAiObjectDeviationResponse> {
        if (!properties.enabled) return emptyList()

        return properties.rules
            .filter { (code, rule) -> rule.enabled && hasDeviation(code, obj) }
            .toList()
            .sortedWith(compareBy({ it.second.priority }, { it.first.name }))
            .map { (code, rule) ->
                MetaAiObjectDeviationResponse(
                    code = code.name,
                    title = rule.title,
                    description = rule.description,
                    priority = rule.priority
                )
            }
    }

    /**
     * Проверяет выполнение условия конкретного отклонения.
     */
    private fun hasDeviation(
        code: MetaAiObjectDeviationCode,
        obj: InitiativeMetaAiObjectEntity
    ): Boolean {
        return when (code) {
            MetaAiObjectDeviationCode.AI_OBJECT_NAME_NOT_FILLED -> obj.name.isNullOrBlank()
            MetaAiObjectDeviationCode.AI_OBJECT_TYPE_NOT_FILLED -> obj.isAgent == null
        }
    }
}

enum class MetaAiObjectType {
    AGENT,
    LLM_SOLUTION
}

/**
 * Активное отклонение связанного AI-объекта.
 */
data class MetaAiObjectDeviationResponse(
    val code: String,
    val title: String,
    val description: String,
    val priority: Int
)

/**
 * Данные связанного AI-объекта для табличного отображения.
 */
data class MetaAiObjectResponse(
    val id: Long,
    val ci: String?,
    val name: String?,
    val type: MetaAiObjectType?,
    val gigaUsageId: String?,
    val metaLink: String?,
    val deviationCount: Int,
    val deviations: List<MetaAiObjectDeviationResponse>
)

/**
 * Список активных AI-объектов инициативы.
 */
data class MetaAiObjectsResponse(
    val objects: List<MetaAiObjectResponse>
)
```
