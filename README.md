```java

@Query("""
    SELECT obj
    FROM InitiativeMetaAiObjectEntity obj
    WHERE obj.initiative.id = :initiativeId
      AND obj.ciSolution IS NOT NULL
""")
fun findAllWithCiSolutionByInitiativeId(@Param("initiativeId") initiativeId: Long): List<InitiativeMetaAiObjectEntity>


/**
 * Обновляет КЭ инициативы в initiative_meta_ai_object.
 */
@Component
class CiSolutionsUpdater(
    private val initiativeMetaAiObjectRepository: InitiativeMetaAiObjectRepository,
    private val userInfoProvider: UserInfoProvider
) {

    fun updateCiSolutions(agent: AIAgentEntity, rawCiSolutionsValue: Any?) {
        val initiativeId = requireNotNull(agent.id) { "Не указан ID инициативы" }
        val currentUserId = userInfoProvider.currentUser().id.toString()
        val requestedCiValues = parseCiSolutions(rawCiSolutionsValue)

        val existingObjects = initiativeMetaAiObjectRepository.findAllWithCiSolutionByInitiativeId(initiativeId)
        val existingByCi = existingObjects.associateBy { it.ciSolution }

        requestedCiValues.forEach { ciValue ->
            val existing = existingByCi[ciValue]

            when {
                existing == null -> createNewCiSolution(agent, ciValue, currentUserId)
                !existing.isActive -> {
                    existing.isActive = true
                    existing.source = InitiativeMetaAiObjectSource.MANUAL
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

    private fun createNewCiSolution(agent: AIAgentEntity, ciValue: String, currentUserId: String) {
        val newObject = InitiativeMetaAiObjectEntity(
            initiative = agent,
            ciSolution = ciValue,
            source = InitiativeMetaAiObjectSource.MANUAL,
            isActive = true,
            createdBy = currentUserId
        )

        initiativeMetaAiObjectRepository.save(newObject)
    }

    private fun parseCiSolutions(rawCiSolutionsValue: Any?): Set<String> {
        if (rawCiSolutionsValue == null) return emptySet()

        val rawList = rawCiSolutionsValue as? List<*>
            ?: throw ResponseStatusException(HttpStatus.BAD_REQUEST, "ciSolutions должен быть массивом")

        return rawList.mapNotNull { value ->
            when (value) {
                is String -> value.trim().takeIf { it.isNotEmpty() }
                is Number -> value.toString()
                else -> throw ResponseStatusException(HttpStatus.BAD_REQUEST, "Недопустимое значение ciSolutions")
            }
        }.toSet()
    }
}
```
