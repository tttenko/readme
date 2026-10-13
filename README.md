```java


data class FinancialEffectsResponse(
    val initiativeId: Long,
    val effects: List<FinancialEffectWithYearResponse>
)

data class FinancialEffectWithYearResponse(
    val id: Long,
    val typeCode: String,
    val typeName: String,
    val year: Int,
    val unit: String,
    val plan: String,
    val fact: String?,
    val targetMonth: String,
    val approvalLinks: List<String>,
    val isPlanAchieved: Boolean,
    val createdAt: Instant
)

data class FinancialEffectTypesResponse(
    val items: List<FinancialEffectTypeItemResponse>
)

data class FinancialEffectTypeItemResponse(
    val code: String,
    val name: String,
    val unit: String
)

@Query(
    """
    SELECT financialEffect
    FROM InitiativeFinancialEffectEntity financialEffect
    JOIN FETCH financialEffect.type
    WHERE financialEffect.initiative.id = :initiativeId
      AND financialEffect.effectYear IN :years
      AND financialEffect.disabled = false
    ORDER BY financialEffect.created ASC, financialEffect.id ASC
    """
)
fun findActiveByInitiativeIdAndYears(
    @Param("initiativeId") initiativeId: Long,
    @Param("years") years: Collection<Int>
): List<InitiativeFinancialEffectEntity>

import org.springframework.data.repository.findByIdOrNull
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.math.BigDecimal
import java.text.MessageFormat
import java.time.LocalDate
import java.time.YearMonth
import java.time.ZoneId
import java.time.Instant

@Service
class FinancialEffectReadService(
    private val aiAgentRepository: AIAgentRepository,
    private val financialEffectRepository: InitiativeFinancialEffectRepository,
    private val financialEffectTypeRepository: FinancialEffectTypeRepository,
    private val messageProvider: MessageProvider
) {

    companion object {
        private val BUSINESS_ZONE = ZoneId.of("Europe/Moscow")
    }

    @Transactional(readOnly = true)
    @OperationDetails("Process 'get financial effects'")
    fun getFinancialEffects(initiativeId: Long, year: Int? = null): FinancialEffectsResponse {
        validateInitiativeExists(initiativeId)

        val selectedYears = getSelectedYears(year)
        val financialEffects = financialEffectRepository.findActiveByInitiativeIdAndYears(initiativeId, selectedYears)

        return FinancialEffectsResponse(
            initiativeId = initiativeId,
            effects = financialEffects.map { buildFinancialEffectResponse(it) }
        )
    }

    @Transactional(readOnly = true)
    @OperationDetails("Process 'get financial effect types'")
    fun getFinancialEffectTypes(): FinancialEffectTypesResponse {
        val financialEffectTypes = financialEffectTypeRepository.findAll()

        return FinancialEffectTypesResponse(
            items = financialEffectTypes.map { financialEffectType ->
                FinancialEffectTypeItemResponse(
                    code = financialEffectType.code,
                    name = financialEffectType.name,
                    unit = financialEffectType.unit
                )
            }
        )
    }

    private fun validateInitiativeExists(initiativeId: Long) {
        aiAgentRepository.findByIdOrNull(initiativeId)
            ?.takeIf { it.disabled != true }
            ?: throw AiNotFoundException(
                errorCode = Metadata.ErrorMessages.INITIATIVE_NOT_FOUND,
                message = messageProvider[Metadata.ErrorMessages.INITIATIVE_NOT_FOUND]
            )
    }

    private fun getSelectedYears(year: Int?): List<Int> {
        if (year != null) {
            if (year < 1) {
                throw AiBadRequestException(
                    errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_READ_YEAR,
                    message = MessageFormat.format(
                        messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_READ_YEAR], year
                    )
                )
            }

            return listOf(year)
        }

        val today = LocalDate.now(BUSINESS_ZONE)
        val currentYear = today.year

        return if (today.monthValue >= 9) {
            listOf(currentYear, currentYear + 1)
        } else {
            listOf(currentYear)
        }
    }

    private fun buildFinancialEffectResponse(
        financialEffect: InitiativeFinancialEffectEntity
    ): FinancialEffectWithYearResponse {
        val financialEffectType = financialEffect.type!!
        val planValue = financialEffect.planValue!!
        val targetMonth = financialEffect.targetMonth!!
        val created = financialEffect.created!!

        return FinancialEffectWithYearResponse(
            id = financialEffect.id,
            typeCode = financialEffectType.code,
            typeName = financialEffectType.name,
            year = financialEffect.effectYear,
            unit = financialEffectType.unit,
            plan = planValue.toPlainString(),
            fact = financialEffect.factValue?.toPlainString(),
            targetMonth = YearMonth.from(targetMonth).toString(),
            approvalLinks = financialEffect.approvalLinks?.map { it.asText() }.orEmpty(),
            isPlanAchieved = calculatePlanAchievement(
                planValue, financialEffect.factValue, financialEffectType
            ),
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

 @GetMapping("/{initiativeId}/financial-effects")
    @Operation(
        summary = "Получение финансовых эффектов инициативы",
        responses = [
            ApiResponse(
                responseCode = "200",
                content = [
                    Content(
                        mediaType = "application/json",
                        schema = Schema(implementation = FinancialEffectsResponse::class)
                    )
                ]
            )
        ]
    )
    @PlatformAudit(
        dataType = AuditParams.DataType.GetFinancialEffectTypes,
        actionType = AuditParams.ActionType.Get,
        value = "Получение финансовых эффектов инициативы"
    )
    @ExceptionApiResponses
    @PreAuthorize("hasAnyAuthority('PROJECT_OFFICE', 'CMS_ADMIN', 'TRANSFORMATION_OFFICE')")
    fun getFinancialEffects(
        @PathVariable initiativeId: Long
    ): FinancialEffectsResponse {
        return financialEffectReadService.getFinancialEffects(initiativeId)
    }

    @GetMapping("/{initiativeId}/financial-effects/{year}")
    @Operation(
        summary = "Получение финансовых эффектов инициативы за выбранный год",
        responses = [
            ApiResponse(
                responseCode = "200",
                content = [
                    Content(
                        mediaType = "application/json",
                        schema = Schema(implementation = FinancialEffectsResponse::class)
                    )
                ]
            )
        ]
    )
    @PlatformAudit(
        dataType = AuditParams.DataType.GetFinancialEffectTypes,
        actionType = AuditParams.ActionType.Get,
        value = "Получение финансовых эффектов инициативы за выбранный год"
    )
    @ExceptionApiResponses
    @PreAuthorize("hasAnyAuthority('PROJECT_OFFICE', 'CMS_ADMIN', 'TRANSFORMATION_OFFICE')")
    fun getFinancialEffectsByYear(
        @PathVariable initiativeId: Long,
        @PathVariable year: Int
    ): FinancialEffectsResponse {
        return financialEffectReadService.getFinancialEffects(initiativeId, year)
    }

    @GetMapping("/financial-effect/types")
    @Operation(
        summary = "Получение типов финансовых эффектов",
        responses = [
            ApiResponse(
                responseCode = "200",
                content = [
                    Content(
                        mediaType = "application/json",
                        schema = Schema(implementation = FinancialEffectTypesResponse::class)
                    )
                ]
            )
        ]
    )
    @PlatformAudit(
        dataType = AuditParams.DataType.GetFinancialEffectTypes,
        actionType = AuditParams.ActionType.Get,
        value = "Получение актуальных типов финансовых эффектов"
    )
    @ExceptionApiResponses
    @PreAuthorize("hasAnyAuthority('PROJECT_OFFICE', 'CMS_ADMIN', 'TRANSFORMATION_OFFICE')")
    fun getFinancialEffectTypes(): FinancialEffectTypesResponse {
        return financialEffectReadService.getFinancialEffectTypes()
    }

    const val GetFinancialEffectTypes = "get_financial_effect_types"

    const val FINANCIAL_EFFECT_INVALID_READ_YEAR = "financial.effect.invalid.read.year"

    financial.effect.invalid.read.year=Некорректный год финансового эффекта: {0}

```
