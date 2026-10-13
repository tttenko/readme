```java


<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd">

    <changeSet id="rename-initiative-financial-effect-timestamps" author="backend">

        <renameColumn
            tableName="initiative_financial_effect"
            oldColumnName="created_at"
            newColumnName="created"
            columnDataType="TIMESTAMP"/>

        <renameColumn
            tableName="initiative_financial_effect"
            oldColumnName="updated_at"
            newColumnName="updated"
            columnDataType="TIMESTAMP"/>

        <rollback>
            <renameColumn
                tableName="initiative_financial_effect"
                oldColumnName="created"
                newColumnName="created_at"
                columnDataType="TIMESTAMP"/>

            <renameColumn
                tableName="initiative_financial_effect"
                oldColumnName="updated"
                newColumnName="updated_at"
                columnDataType="TIMESTAMP"/>
        </rollback>

    </changeSet>
</databaseChangeLog>

data class CreateFinancialEffectsRequest(
    val effects: List<CreateFinancialEffectItemRequest> = emptyList()
)

data class CreateFinancialEffectItemRequest(
    val typeCode: String? = null,
    val plan: String? = null,
    val fact: String? = null,
    val targetMonth: String? = null,
    val approvalLinks: List<String>? = null
)

data class FinancialEffectResponse(
    val initiativeId: Long,
    val effects: List<FinancialEffectItemResponse>
)

data class FinancialEffectItemResponse(
    val id: Long,
    val typeCode: String,
    val typeName: String,
    val unit: String,
    val plan: String,
    val fact: String?,
    val targetMonth: String,
    val approvalLinks: List<String>,
    val isPlanAchieved: Boolean,
    val createdAt: Instant
)

@Repository
interface InitiativeFinancialEffectRepository : JpaRepository<InitiativeFinancialEffectEntity, Long> {

    @Query(
        """
        SELECT CASE WHEN COUNT(financialEffect) > 0 THEN true ELSE false END
        FROM InitiativeFinancialEffectEntity financialEffect
        WHERE financialEffect.initiative.id = :initiativeId
          AND financialEffect.effectYear = :year
          AND financialEffect.disabled = false
        """
    )
    fun existsActiveByInitiativeIdAndYear(
        @Param("initiativeId") initiativeId: Long,
        @Param("year") year: Int
    ): Boolean

    @Query(
        """
        SELECT financialEffect
        FROM InitiativeFinancialEffectEntity financialEffect
        JOIN FETCH financialEffect.type
        WHERE financialEffect.initiative.id = :initiativeId
          AND financialEffect.effectYear = :year
          AND financialEffect.disabled = false
        ORDER BY financialEffect.created ASC, financialEffect.id ASC
        """
    )
    fun findActiveByInitiativeIdAndYear(
        @Param("initiativeId") initiativeId: Long,
        @Param("year") year: Int
    ): List<InitiativeFinancialEffectEntity>
}

@Repository
interface FinancialEffectTypeRepository : JpaRepository<FinancialEffectTypeEntity, String>

@Service
class FinancialEffectService(
    private val aiAgentRepository: AIAgentRepository,
    private val financialEffectRepository: InitiativeFinancialEffectRepository,
    private val financialEffectTypeRepository: FinancialEffectTypeRepository,
    private val userInfoProvider: UserInfoProvider,
    private val objectMapper: ObjectMapper,
    private val messageProvider: MessageProvider
) {

    companion object {
        private val BUSINESS_ZONE = ZoneId.of("Europe/Moscow")
    }

    @Transactional
    @OperationDetails("Process 'create financial effects'")
    fun createFinancialEffects(
        initiativeId: Long,
        year: Int,
        request: CreateFinancialEffectsRequest
    ): FinancialEffectResponse {
        val initiative = getInitiativeForUpdate(initiativeId)

        validateYear(year)
        validateFinancialEffectsNotCreated(initiativeId, year)
        validateFinancialEffectsNotEmpty(request.effects)

        val financialEffectTypes = getFinancialEffectTypes(request.effects)
        val currentUserId = userInfoProvider.currentUser().id

        val financialEffects = createFinancialEffectEntities(
            initiative, year, request.effects, financialEffectTypes, currentUserId
        )

        financialEffectRepository.saveAllAndFlush(financialEffects)

        return getFinancialEffectResponse(initiativeId, year)
    }

    private fun getInitiativeForUpdate(initiativeId: Long): AIAgentEntity {
        return aiAgentRepository.findByIdForUpdate(initiativeId)
            ?.takeIf { it.disabled != true }
            ?: throw AiNotFoundException(
                errorCode = Metadata.ErrorMessages.INITIATIVE_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.INITIATIVE_NOT_FOUND],
                    initiativeId
                )
            )
    }

    private fun validateYear(year: Int) {
        val today = LocalDate.now(BUSINESS_ZONE)
        val currentYear = today.year

        val isCurrentYear = year == currentYear
        val isNextYearAvailable = today.monthValue >= 9 && year == currentYear + 1

        if (!isCurrentYear && !isNextYearAvailable) {
            throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_YEAR,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_YEAR],
                    year
                )
            )
        }
    }

    private fun validateFinancialEffectsNotCreated(initiativeId: Long, year: Int) {
        if (financialEffectRepository.existsActiveByInitiativeIdAndYear(initiativeId, year)) {
            throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_ALREADY_INITIALIZED,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_ALREADY_INITIALIZED],
                    year
                )
            )
        }
    }

    private fun validateFinancialEffectsNotEmpty(financialEffects: List<CreateFinancialEffectItemRequest>) {
        if (financialEffects.isEmpty()) {
            throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_EMPTY_LIST,
                message = messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_EMPTY_LIST]
            )
        }
    }

    private fun getFinancialEffectTypes(
        financialEffects: List<CreateFinancialEffectItemRequest>
    ): Map<String, FinancialEffectTypeEntity> {
        val typeCodes = financialEffects.mapNotNull { financialEffect ->
            financialEffect.typeCode?.trim()?.takeIf { it.isNotEmpty() }
        }.distinct()

        return financialEffectTypeRepository.findAllById(typeCodes).associateBy { it.code }
    }

    private fun createFinancialEffectEntities(
        initiative: AIAgentEntity,
        year: Int,
        financialEffects: List<CreateFinancialEffectItemRequest>,
        financialEffectTypes: Map<String, FinancialEffectTypeEntity>,
        currentUserId: Long
    ): List<InitiativeFinancialEffectEntity> {
        return financialEffects.mapIndexed { index, financialEffect ->
            val cardNumber = index + 1

            val typeCode = getRequiredField(financialEffect.typeCode, "typeCode", cardNumber)
            val plan = getRequiredField(financialEffect.plan, "plan", cardNumber)
            val targetMonth = getRequiredField(financialEffect.targetMonth, "targetMonth", cardNumber)

            val financialEffectType = financialEffectTypes[typeCode]
                ?: throw AiBadRequestException(
                    errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_TYPE_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_TYPE_NOT_FOUND],
                        typeCode
                    )
                )

            InitiativeFinancialEffectEntity(
                initiative = initiative,
                effectYear = year,
                type = financialEffectType,
                planValue = parseNonNegativeAmount(plan, "plan", cardNumber),
                factValue = financialEffect.fact?.let { parseNonNegativeAmount(it, "fact", cardNumber) },
                targetMonth = parseTargetMonth(targetMonth, cardNumber),
                approvalLinks = objectMapper.valueToTree<JsonNode>(financialEffect.approvalLinks.orEmpty()),
                disabled = false,
                createdBy = currentUserId
            )
        }
    }

    private fun getRequiredField(value: String?, fieldName: String, cardNumber: Int): String {
        return value?.trim()?.takeIf { it.isNotEmpty() }
            ?: throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_REQUIRED_FIELD,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_REQUIRED_FIELD],
                    fieldName, cardNumber
                )
            )
    }

    private fun parseNonNegativeAmount(value: String, fieldName: String, cardNumber: Int): BigDecimal {
        val amount = value.trim().toBigDecimalOrNull()

        if (amount == null || amount.signum() < 0) {
            throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_AMOUNT,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_AMOUNT],
                    fieldName, cardNumber
                )
            )
        }

        return amount
    }

    private fun parseTargetMonth(value: String, cardNumber: Int): LocalDate {
        return try {
            YearMonth.parse(value).atDay(1)
        } catch (exception: DateTimeParseException) {
            throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_TARGET_MONTH,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_TARGET_MONTH],
                    cardNumber
                )
            )
        }
    }

    private fun getFinancialEffectResponse(initiativeId: Long, year: Int): FinancialEffectResponse {
        val financialEffects = financialEffectRepository.findActiveByInitiativeIdAndYear(initiativeId, year)

        return FinancialEffectResponse(
            initiativeId = initiativeId,
            effects = financialEffects.map { buildFinancialEffectResponse(it) }
        )
    }

    private fun buildFinancialEffectResponse(
        financialEffect: InitiativeFinancialEffectEntity
    ): FinancialEffectItemResponse {
        val financialEffectType = financialEffect.type!!
        val planValue = financialEffect.planValue!!
        val targetMonth = financialEffect.targetMonth!!
        val created = financialEffect.created!!

        return FinancialEffectItemResponse(
            id = financialEffect.id,
            typeCode = financialEffectType.code,
            typeName = financialEffectType.name,
            unit = financialEffectType.unit,
            plan = planValue.toPlainString(),
            fact = financialEffect.factValue?.toPlainString(),
            targetMonth = YearMonth.from(targetMonth).toString(),
            approvalLinks = financialEffect.approvalLinks?.map { it.asText() }.orEmpty(),
            isPlanAchieved = calculatePlanAchievement(planValue, financialEffect.factValue, financialEffectType),
            createdAt = created.atZone(ZoneId.systemDefault()).toInstant()
        )
    }

    private fun calculatePlanAchievement(
        planValue: BigDecimal,
        factValue: BigDecimal?,
        financialEffectType: FinancialEffectTypeEntity
    ): Boolean {
        if (factValue == null) return false

        return if (financialEffectType.moreIsBetter) {
            factValue >= planValue
        } else {
            factValue <= planValue
        }
    }
}

@RestController
@RequestMapping("/api/v1/ai-agent")
class FinancialEffectController(
    private val financialEffectService: FinancialEffectService
) {

    @PostMapping("/{initiativeId}/financial-effects/{year}")
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(
        summary = "Первичное создание финансовых эффектов инициативы",
        responses = [
            ApiResponse(
                responseCode = "201",
                content = [
                    Content(
                        mediaType = "application/json",
                        schema = Schema(implementation = FinancialEffectResponse::class)
                    )
                ]
            )
        ]
    )
    @PlatformAudit(
        dataType = AuditParams.DataType.CreateFinancialEffects,
        actionType = AuditParams.ActionType.Create,
        value = "Создание финансовых эффектов инициативы"
    )
    @ExceptionApiResponses
    @PreAuthorize("hasAnyAuthority('PROJECT_OFFICE', 'CMS_ADMIN', 'TRANSFORMATION_OFFICE')")
    fun createFinancialEffects(
        @PathVariable initiativeId: Long,
        @PathVariable year: Int,
        @Valid @RequestBody request: CreateFinancialEffectsRequest
    ): FinancialEffectResponse {
        return financialEffectService.createFinancialEffects(initiativeId, year, request)
    }

const val CreateFinancialEffects = "create_financial_effect_types"

const val FINANCIAL_EFFECT_ALREADY_INITIALIZED = "financial.effect.already.initialized"
const val FINANCIAL_EFFECT_EMPTY_LIST = "financial.effect.empty.list"
const val FINANCIAL_EFFECT_INVALID_YEAR = "financial.effect.invalid.year"
const val FINANCIAL_EFFECT_TYPE_NOT_FOUND = "financial.effect.type.not.found"
const val FINANCIAL_EFFECT_REQUIRED_FIELD = "financial.effect.required.field"
const val FINANCIAL_EFFECT_INVALID_AMOUNT = "financial.effect.invalid.amount"
const val FINANCIAL_EFFECT_INVALID_TARGET_MONTH = "financial.effect.invalid.target.month"

financial.effect.already.initialized=Финансовые эффекты за {0} год уже существуют
financial.effect.empty.list=Необходимо передать хотя бы одну карточку финансового эффекта
financial.effect.invalid.year=Создание финансовых эффектов за {0} год недоступно
financial.effect.type.not.found=Тип финансового эффекта {0} не найден
financial.effect.required.field=Поле {0} обязательно для карточки {1}
financial.effect.invalid.amount=Поле {0} карточки {1} должно содержать неотрицательное число
financial.effect.invalid.target.month=Некорректный срок достижения в карточке {0}. Ожидается формат YYYY-MM

```
