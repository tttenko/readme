```java
import org.springframework.data.repository.findByIdOrNull
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

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
    fun getMetaAiObjects(initiativeId: Long): List<MetaAiObjectResponse> {
        if (initiativeId <= 0) {
            throw AiBadRequestException(
                errorCode = IncorrectInputValue,
                message = messageProvider[IncorrectInputValue]
            )
        }

        aiAgentRepository.findByIdOrNull(initiativeId)
            ?.takeIf { it.disabled != true }
            ?: throw AiNotFoundException(
                errorCode = INITIATIVE_NOT_FOUND,
                message = messageProvider[INITIATIVE_NOT_FOUND]
            )

        val crossgoal = jiraIssueRepository.findCrossgoalKeysByInitiativeId(initiativeId)
            .firstOrNull()
            ?: return emptyList()

        val objects = metaAiObjectRepository.findActiveByInitiativeIdAndCrossgoal(
            initiativeId = initiativeId,
            crossgoal = crossgoal
        )

        return objects.map { obj ->
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
    }
}

```
