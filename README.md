```java


import org.springframework.stereotype.Component

/**
 * Обновляет связанные AI-объекты инициативы записями КЭ (initiative_meta_ai_object).
 *
 * Новые значения добавляются с source = MANUAL, ранее деактивированные значения
 * реактивируются, отсутствующие в запросе записи деактивируются.
 */
@Component
open class CiSolutionsUpdater(
    private val initiativeMetaAiObjectRepository: InitiativeMetaAiObjectRepository,
    private val userInfoProvider: UserInfoProvider
) {

    /**
     * Синхронизирует КЭ инициативы с переданным списком значений.
     *
     * @param agent сущность инициативы
     * @param rawCiSolutionsValue входное значение (список строк или null)
     */
    fun updateCiSolutions(agent: AIAgentEntity, rawCiSolutionsValue: Any?) {
        val requestedCiValues = parseCiSolutions(rawCiSolutionsValue)
        val existingObjects = initiativeMetaAiObjectRepository.findAllWithCiSolutionByInitiativeId(agent.id)
        val existingByCi = existingObjects.associateBy { it.ciSolution }
        val currentUserId = userInfoProvider.currentUser().id.toString()

        val crossgoal by lazy { getCrossgoal(agent) }

        requestedCiValues.forEach { ciValue ->
            val existing = existingByCi[ciValue]

            when {
                existing == null -> createNewCiSolution(agent, crossgoal, ciValue, currentUserId)
                !existing.isActive -> {
                    existing.isActive = true
                    existing.updatedBy = currentUserId
                }
            }
        }

        existingObjects.forEach { obj ->
            if (obj.isActive && obj.ciSolution !in requestedCiValues) {
                obj.isActive = false
                obj.updatedBy = currentUserId
            }
        }
    }

    /**
     * Создаёт новую связь КЭ с инициативой.
     */
    private fun createNewCiSolution(agent: AIAgentEntity, crossgoal: String, ciValue: String, currentUserId: String) {
        val newObject = InitiativeMetaAiObjectEntity(
            initiative = agent,
            crossgoal = crossgoal,
            ciSolution = ciValue,
            source = InitiativeMetaAiObjectSource.MANUAL,
            isActive = true,
            createdBy = currentUserId
        )

        initiativeMetaAiObjectRepository.save(newObject)
    }

    /**
     * Получает CROSSGOAL инициативы из связанной записи Jira.
     */
    private fun getCrossgoal(agent: AIAgentEntity): String {
        val crossgoalKeys = agent.jiraIssues
            .filter { it.project.equals("crossgoal", ignoreCase = true) && it.type.equals("initiative", ignoreCase = true) }
            .mapNotNull { it.jiraKey?.trim()?.takeIf(String::isNotEmpty) }
            .distinct()

        check(crossgoalKeys.size == 1) {
            "Для инициативы ${agent.id} должен существовать ровно один CROSSGOAL, найдено: ${crossgoalKeys.size}"
        }

        return crossgoalKeys.single()
    }

    /**
     * Преобразует входное значение в уникальный набор строк.
     */
    private fun parseCiSolutions(rawCiSolutionsValue: Any?): Set<String> {
        if (rawCiSolutionsValue == null) return emptySet()
        val rawList = rawCiSolutionsValue as? List<*> ?: return emptySet()

        return rawList
            .mapNotNull { value ->
                when (value) {
                    is String -> value.trim().takeIf { it.isNotEmpty() }
                    is Number -> value.toString()
                    else -> null
                }
            }.toSet()
    }
}

```
