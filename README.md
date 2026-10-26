```java
@Component
class JiraInitiativeFinancialEffectSynchronizer(
    private val financialEffectRepository: InitiativeFinancialEffectRepository,
    private val financialEffectTypeRepository: FinancialEffectTypeRepository,
    private val numericValueParser: JiraNumericValueParser,
    private val objectMapper: ObjectMapper,
) {

    companion object {
        private const val INCOME_FINANCIAL_EFFECT = "INCOME_FINANCIAL_EFFECT"
        private const val STAFF_REDUCTIONS_FINANCIAL_EFFECT = "STAFF_REDUCTIONS_FINANCIAL_EFFECT"

        private const val SYSTEM_ACTOR = 0L

        private val BUSINESS_ZONE = ZoneId.of("Europe/Moscow")

        private val FINANCIAL_EFFECT_TYPE_CODES = listOf(
            INCOME_FINANCIAL_EFFECT,
            STAFF_REDUCTIONS_FINANCIAL_EFFECT,
        )

        private val log by logger()
    }

    /**
     * Создаёт финансовые эффекты для новой инициативы.
     *
     * customfield_30401 -> INCOME_FINANCIAL_EFFECT
     * customfield_34300 -> STAFF_REDUCTIONS_FINANCIAL_EFFECT
     */
    fun createFinancialEffects(
        initiative: AIAgentEntity,
        jiraKey: String?,
        revenueValue: String?,
        optimizationValue: String?,
    ) {
        val currentYear = LocalDate.now(BUSINESS_ZONE).year
        val financialEffectTypes = getFinancialEffectTypes()

        val financialEffects = listOfNotNull(
            createFinancialEffect(
                initiative = initiative,
                jiraKey = jiraKey,
                jiraFieldName = "customfield_30401",
                jiraFieldValue = revenueValue,
                typeCode = INCOME_FINANCIAL_EFFECT,
                currentYear = currentYear,
                financialEffectTypes = financialEffectTypes,
            ),
            createFinancialEffect(
                initiative = initiative,
                jiraKey = jiraKey,
                jiraFieldName = "customfield_34300",
                jiraFieldValue = optimizationValue,
                typeCode = STAFF_REDUCTIONS_FINANCIAL_EFFECT,
                currentYear = currentYear,
                financialEffectTypes = financialEffectTypes,
            ),
        )

        if (financialEffects.isEmpty()) {
            log.debug(
                "No financial effects found for new Jira initiative: jiraKey={}, agentId={}",
                jiraKey,
                initiative.id,
            )
            return
        }

        financialEffectRepository.saveAll(financialEffects)

        log.debug(
            "Created financial effects from Jira: jiraKey={}, agentId={}, typeCodes={}",
            jiraKey,
            initiative.id,
            financialEffects.mapNotNull { financialEffect -> financialEffect.type?.code },
        )
    }

    /**
     * Синхронизирует финансовые эффекты существующей инициативы.
     *
     * Если системная карточка текущего года уже существует — обновляет её.
     * Если системной карточки нет — создаёт новую.
     * Если Jira-поле пустое — ничего не меняет.
     */
    fun synchronizeFinancialEffects(
        initiative: AIAgentEntity,
        jiraKey: String?,
        revenueValue: String?,
        optimizationValue: String?,
    ) {
        val currentYear = LocalDate.now(BUSINESS_ZONE).year

        val existingFinancialEffects = financialEffectRepository
            .findActiveByInitiativeIdAndYear(
                initiativeId = initiative.id,
                year = currentYear,
            )

        val financialEffectTypes = getFinancialEffectTypes()

        synchronizeFinancialEffect(
            initiative = initiative,
            jiraKey = jiraKey,
            jiraFieldName = "customfield_30401",
            jiraFieldValue = revenueValue,
            typeCode = INCOME_FINANCIAL_EFFECT,
            currentYear = currentYear,
            existingFinancialEffects = existingFinancialEffects,
            financialEffectTypes = financialEffectTypes,
        )

        synchronizeFinancialEffect(
            initiative = initiative,
            jiraKey = jiraKey,
            jiraFieldName = "customfield_34300",
            jiraFieldValue = optimizationValue,
            typeCode = STAFF_REDUCTIONS_FINANCIAL_EFFECT,
            currentYear = currentYear,
            existingFinancialEffects = existingFinancialEffects,
            financialEffectTypes = financialEffectTypes,
        )
    }

    private fun createFinancialEffect(
        initiative: AIAgentEntity,
        jiraKey: String?,
        jiraFieldName: String,
        jiraFieldValue: String?,
        typeCode: String,
        currentYear: Int,
        financialEffectTypes: Map<String, FinancialEffectTypeEntity>,
    ): InitiativeFinancialEffectEntity? {
        val planValue = parseFinancialEffect(
            jiraKey = jiraKey,
            jiraFieldName = jiraFieldName,
            jiraFieldValue = jiraFieldValue,
        ) ?: return null

        val financialEffectType = requireNotNull(financialEffectTypes[typeCode]) {
            "Financial effect type '$typeCode' is not configured"
        }

        return InitiativeFinancialEffectEntity(
            initiative = initiative,
            effectYear = currentYear,
            type = financialEffectType,
            planValue = planValue,
            factValue = null,
            targetMonth = LocalDate.of(currentYear, 12, 31),
            approvalLinks = objectMapper.createArrayNode(),
            disabled = false,
            createdBy = SYSTEM_ACTOR,
            updatedBy = SYSTEM_ACTOR,
        )
    }

    private fun synchronizeFinancialEffect(
        initiative: AIAgentEntity,
        jiraKey: String?,
        jiraFieldName: String,
        jiraFieldValue: String?,
        typeCode: String,
        currentYear: Int,
        existingFinancialEffects: List<InitiativeFinancialEffectEntity>,
        financialEffectTypes: Map<String, FinancialEffectTypeEntity>,
    ) {
        val planValue = parseFinancialEffect(
            jiraKey = jiraKey,
            jiraFieldName = jiraFieldName,
            jiraFieldValue = jiraFieldValue,
        ) ?: return

        val financialEffectsByType = existingFinancialEffects
            .filter { financialEffect ->
                financialEffect.type?.code == typeCode
            }

        val existingSystemFinancialEffect = financialEffectsByType
            .firstOrNull { financialEffect ->
                financialEffect.createdBy == SYSTEM_ACTOR
            }

        val existingFinancialEffect = existingSystemFinancialEffect
            ?: financialEffectsByType.singleOrNull()

        if (existingFinancialEffect != null) {
            existingFinancialEffect.apply {
                this.planValue = planValue
                factValue = null
                targetMonth = LocalDate.of(currentYear, 12, 31)
                approvalLinks = objectMapper.createArrayNode()
                disabled = false
                updatedBy = SYSTEM_ACTOR
            }

            financialEffectRepository.save(existingFinancialEffect)

            log.debug(
                "Updated financial effect from Jira: jiraKey={}, agentId={}, typeCode={}, planValue={}",
                jiraKey,
                initiative.id,
                typeCode,
                planValue,
            )

            return
        }

        if (financialEffectsByType.size > 1) {
            log.warn(
                "Multiple financial effects found for Jira synchronization: " +
                    "jiraKey={}, agentId={}, typeCode={}, existingIds={}",
                jiraKey,
                initiative.id,
                typeCode,
                financialEffectsByType.map { financialEffect -> financialEffect.id },
            )
        }

        val financialEffectType = requireNotNull(financialEffectTypes[typeCode]) {
            "Financial effect type '$typeCode' is not configured"
        }

        val financialEffect = InitiativeFinancialEffectEntity(
            initiative = initiative,
            effectYear = currentYear,
            type = financialEffectType,
            planValue = planValue,
            factValue = null,
            targetMonth = LocalDate.of(currentYear, 12, 31),
            approvalLinks = objectMapper.createArrayNode(),
            disabled = false,
            createdBy = SYSTEM_ACTOR,
            updatedBy = SYSTEM_ACTOR,
        )

        financialEffectRepository.save(financialEffect)

        log.debug(
            "Created financial effect from Jira synchronization: jiraKey={}, agentId={}, typeCode={}, planValue={}",
            jiraKey,
            initiative.id,
            typeCode,
            planValue,
        )
    }

    private fun parseFinancialEffect(
        jiraKey: String?,
        jiraFieldName: String,
        jiraFieldValue: String?,
    ): BigDecimal? {
        if (jiraFieldValue.isNullOrBlank()) {
            return null
        }

        val financialEffect = numericValueParser.parseFirst(jiraFieldValue)

        if (financialEffect == null) {
            log.warn(
                "Cannot parse Jira financial effect: jiraKey={}, field={}, value={}",
                jiraKey,
                jiraFieldName,
                jiraFieldValue,
            )
        }

        return financialEffect
    }

    private fun getFinancialEffectTypes(): Map<String, FinancialEffectTypeEntity> {
        val financialEffectTypes = financialEffectTypeRepository
            .findAllById(FINANCIAL_EFFECT_TYPE_CODES)
            .associateBy { financialEffectType ->
                financialEffectType.code
            }

        val missingTypeCodes = FINANCIAL_EFFECT_TYPE_CODES
            .filterNot(financialEffectTypes::containsKey)

        check(missingTypeCodes.isEmpty()) {
            "Financial effect types are not configured: ${missingTypeCodes.joinToString()}"
        }

        return financialEffectTypes
    }
}
```
