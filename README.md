```java


data class UpdateFinancialEffectsRequest(
    val effects: List<UpdateFinancialEffectItemRequest>
)

data class UpdateFinancialEffectItemRequest(
    val id: Long? = null,
    val typeCode: String? = null,
    val plan: String? = null,
    val fact: String? = null,
    val targetMonth: String? = null,
    val approvalLinks: List<String>? = null
)

@Query(
    """
    SELECT CASE WHEN COUNT(financialEffect) > 0 THEN true ELSE false END
    FROM InitiativeFinancialEffectEntity financialEffect
    WHERE financialEffect.initiative.id = :initiativeId
      AND financialEffect.effectYear = :year
    """
)
fun existsAnyByInitiativeIdAndYear(
    @Param("initiativeId") initiativeId: Long,
    @Param("year") year: Int
): Boolean

@Service
class FinancialEffectUpdateService(
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

    private data class ValidatedFinancialEffect(
        val id: Long?,
        val type: FinancialEffectTypeEntity,
        val planValue: BigDecimal,
        val factValue: BigDecimal?,
        val targetMonth: LocalDate,
        val approvalLinks: JsonNode
    )

    @Transactional
    @OperationDetails("Process 'update financial effects'")
    fun updateFinancialEffects(
        initiativeId: Long,
        year: Int,
        request: UpdateFinancialEffectsRequest
    ): FinancialEffectResponse {
        val initiative = getInitiativeForUpdate(initiativeId)

        validateYear(year)
        validateFinancialEffectsInitialized(initiativeId, year)

        val existingFinancialEffects = financialEffectRepository.findActiveByInitiativeIdAndYear(initiativeId, year)
        val existingFinancialEffectsById = existingFinancialEffects.associateBy { it.id }

        val financialEffects = validateFinancialEffects(request.effects, year, existingFinancialEffectsById)
        val currentUserId = userInfoProvider.currentUser().id

        val newFinancialEffects = saveFinancialEffects(
            initiative, year, financialEffects, existingFinancialEffectsById, currentUserId
        )

        disableRemovedFinancialEffects(existingFinancialEffects, financialEffects, currentUserId)

        if (newFinancialEffects.isNotEmpty()) {
            financialEffectRepository.saveAll(newFinancialEffects)
        }

        financialEffectRepository.flush()

        return getFinancialEffectResponse(initiativeId, year)
    }

    private fun getInitiativeForUpdate(initiativeId: Long): AIAgentEntity {
        return aiAgentRepository.findByIdForUpdate(initiativeId)
            ?.takeIf { it.disabled != true }
            ?: throw AiNotFoundException(
                errorCode = Metadata.ErrorMessages.INITIATIVE_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.INITIATIVE_NOT_FOUND], initiativeId
                )
            )
    }

    private fun validateYear(year: Int) {
        val today = LocalDate.now(BUSINESS_ZONE)
        val currentYear = today.year

        val isCurrentOrPastYear = year in 1..currentYear
        val isNextYearAvailable = today.monthValue >= 9 && year == currentYear + 1

        if (!isCurrentOrPastYear && !isNextYearAvailable) {
            throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_YEAR,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_YEAR], year
                )
            )
        }
    }

    private fun validateFinancialEffectsInitialized(initiativeId: Long, year: Int) {
        if (!financialEffectRepository.existsAnyByInitiativeIdAndYear(initiativeId, year)) {
            throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_NOT_INITIALIZED,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_NOT_INITIALIZED], year
                )
            )
        }
    }

    private fun validateFinancialEffects(
        requestedFinancialEffects: List<UpdateFinancialEffectItemRequest>,
        year: Int,
        existingFinancialEffectsById: Map<Long, InitiativeFinancialEffectEntity>
    ): List<ValidatedFinancialEffect> {
        val requestedFinancialEffectIds = requestedFinancialEffects.mapNotNull { it.id }
        val duplicateFinancialEffectId = requestedFinancialEffectIds.groupingBy { it }.eachCount()
            .entries.firstOrNull { it.value > 1 }?.key

        if (duplicateFinancialEffectId != null) {
            throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_DUPLICATE_ID,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_DUPLICATE_ID],
                    duplicateFinancialEffectId
                )
            )
        }

        val typeCodes = requestedFinancialEffects.mapNotNull { it.typeCode?.trim()?.takeIf(String::isNotEmpty) }.distinct()
        val financialEffectTypes = financialEffectTypeRepository.findAllById(typeCodes).associateBy { it.code }
        val currentYear = LocalDate.now(BUSINESS_ZONE).year

        return requestedFinancialEffects.mapIndexed { index, requestedFinancialEffect ->
            val cardNumber = index + 1

            val typeCode = getRequiredField(requestedFinancialEffect.typeCode, "typeCode", cardNumber)
            val plan = getRequiredField(requestedFinancialEffect.plan, "plan", cardNumber)
            val targetMonth = getRequiredField(requestedFinancialEffect.targetMonth, "targetMonth", cardNumber)

            val financialEffectType = financialEffectTypes[typeCode]
                ?: throw AiBadRequestException(
                    errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_TYPE_NOT_FOUND,
                    message = MessageFormat.format(
                        messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_TYPE_NOT_FOUND], typeCode
                    )
                )

            if (requestedFinancialEffect.id == null && year < currentYear) {
                throw AiBadRequestException(
                    errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_PAST_YEAR_CREATE_FORBIDDEN,
                    message = MessageFormat.format(
                        messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_PAST_YEAR_CREATE_FORBIDDEN], year
                    )
                )
            }

            if (requestedFinancialEffect.id != null) {
                val existingFinancialEffect = existingFinancialEffectsById[requestedFinancialEffect.id]
                    ?: throw AiBadRequestException(
                        errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_NOT_FOUND,
                        message = MessageFormat.format(
                            messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_NOT_FOUND],
                            requestedFinancialEffect.id
                        )
                    )

                if (existingFinancialEffect.type?.code != typeCode) {
                    throw AiBadRequestException(
                        errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_IMMUTABLE_TYPE,
                        message = MessageFormat.format(
                            messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_IMMUTABLE_TYPE],
                            requestedFinancialEffect.id
                        )
                    )
                }
            }

            ValidatedFinancialEffect(
                id = requestedFinancialEffect.id,
                type = financialEffectType,
                planValue = parseNonNegativeAmount(plan, "plan", cardNumber),
                factValue = requestedFinancialEffect.fact?.let { parseNonNegativeAmount(it, "fact", cardNumber) },
                targetMonth = parseTargetMonth(targetMonth, cardNumber),
                approvalLinks = objectMapper.valueToTree<JsonNode>(requestedFinancialEffect.approvalLinks.orEmpty())
            )
        }
    }

    private fun saveFinancialEffects(
        initiative: AIAgentEntity,
        year: Int,
        financialEffects: List<ValidatedFinancialEffect>,
        existingFinancialEffectsById: Map<Long, InitiativeFinancialEffectEntity>,
        currentUserId: Long
    ): List<InitiativeFinancialEffectEntity> {
        val newFinancialEffects = mutableListOf<InitiativeFinancialEffectEntity>()

        financialEffects.forEach { financialEffect ->
            if (financialEffect.id == null) {
                newFinancialEffects.add(
                    InitiativeFinancialEffectEntity(
                        initiative = initiative,
                        effectYear = year,
                        type = financialEffect.type,
                        planValue = financialEffect.planValue,
                        factValue = financialEffect.factValue,
                        targetMonth = financialEffect.targetMonth,
                        approvalLinks = financialEffect.approvalLinks,
                        disabled = false,
                        createdBy = currentUserId
                    )
                )
            } else {
                val existingFinancialEffect = existingFinancialEffectsById.getValue(financialEffect.id)
                updateExistingFinancialEffect(existingFinancialEffect, financialEffect, currentUserId)
            }
        }

        return newFinancialEffects
    }

    private fun updateExistingFinancialEffect(
        existingFinancialEffect: InitiativeFinancialEffectEntity,
        financialEffect: ValidatedFinancialEffect,
        currentUserId: Long
    ) {
        val planChanged = amountsDiffer(existingFinancialEffect.planValue, financialEffect.planValue)
        val factChanged = amountsDiffer(existingFinancialEffect.factValue, financialEffect.factValue)
        val targetMonthChanged = existingFinancialEffect.targetMonth != financialEffect.targetMonth
        val approvalLinksChanged = existingFinancialEffect.approvalLinks != financialEffect.approvalLinks

        if (!planChanged && !factChanged && !targetMonthChanged && !approvalLinksChanged) return

        existingFinancialEffect.planValue = financialEffect.planValue
        existingFinancialEffect.factValue = financialEffect.factValue
        existingFinancialEffect.targetMonth = financialEffect.targetMonth
        existingFinancialEffect.approvalLinks = financialEffect.approvalLinks
        existingFinancialEffect.updatedBy = currentUserId
    }

    private fun amountsDiffer(currentValue: BigDecimal?, newValue: BigDecimal?): Boolean {
        if (currentValue == null || newValue == null) return currentValue != newValue

        return currentValue.compareTo(newValue) != 0
    }

    private fun disableRemovedFinancialEffects(
        existingFinancialEffects: List<InitiativeFinancialEffectEntity>,
        financialEffects: List<ValidatedFinancialEffect>,
        currentUserId: Long
    ) {
        val retainedFinancialEffectIds = financialEffects.mapNotNull { it.id }.toSet()

        existingFinancialEffects.filter { it.id !in retainedFinancialEffectIds }.forEach { financialEffect ->
            financialEffect.disabled = true
            financialEffect.updatedBy = currentUserId
        }
    }

    private fun getRequiredField(value: String?, fieldName: String, cardNumber: Int): String {
        return value?.trim()?.takeIf { it.isNotEmpty() }
            ?: throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_REQUIRED_FIELD,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_REQUIRED_FIELD], fieldName, cardNumber
                )
            )
    }

    private fun parseNonNegativeAmount(value: String, fieldName: String, cardNumber: Int): BigDecimal {
        val amount = value.trim().toBigDecimalOrNull()

        if (amount == null || amount.signum() < 0) {
            throw AiBadRequestException(
                errorCode = Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_AMOUNT,
                message = MessageFormat.format(
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_AMOUNT], fieldName, cardNumber
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
                    messageProvider[Metadata.ErrorMessages.FINANCIAL_EFFECT_INVALID_TARGET_MONTH], cardNumber
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

@PutMapping("/{initiativeId}/financial-effects/{year}")
@ResponseStatus(HttpStatus.OK)
@Operation(
    summary = "Сохранение финансовых эффектов инициативы",
    responses = [
        ApiResponse(
            responseCode = "200",
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
    dataType = AuditParams.DataType.UpdateFinancialEffects,
    actionType = AuditParams.ActionType.Update,
    value = "Изменение финансовых эффектов инициативы"
)
@ExceptionApiResponses
@PreAuthorize("hasAnyAuthority('PROJECT_OFFICE', 'CMS_ADMIN', 'TRANSFORMATION_OFFICE')")
fun updateFinancialEffects(
    @PathVariable initiativeId: Long,
    @PathVariable year: Int,
    @RequestBody request: UpdateFinancialEffectsRequest
): FinancialEffectResponse {
    return financialEffectUpdateService.updateFinancialEffects(initiativeId, year, request)
}

const val UpdateFinancialEffects = "put_financial_effect_types"

const val FINANCIAL_EFFECT_NOT_INITIALIZED = "financial.effect.not.initialized"
const val FINANCIAL_EFFECT_NOT_FOUND = "financial.effect.not.found"
const val FINANCIAL_EFFECT_DUPLICATE_ID = "financial.effect.duplicate.id"
const val FINANCIAL_EFFECT_IMMUTABLE_TYPE = "financial.effect.immutable.type"
const val FINANCIAL_EFFECT_PAST_YEAR_CREATE_FORBIDDEN = "financial.effect.past.year.create.forbidden"

financial.effect.not.initialized=Финансовые эффекты за {0} год ещё не были созданы
financial.effect.not.found=Финансовый эффект с ID {0} не найден среди активных карточек инициативы за выбранный год
financial.effect.duplicate.id=Финансовый эффект с ID {0} передан несколько раз
financial.effect.immutable.type=Нельзя изменять тип сохранённого финансового эффекта с ID {0}
financial.effect.past.year.create.forbidden=Создание новых финансовых эффектов за {0} год запрещено


```
