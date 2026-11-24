```java
/**
     * Возвращает историю изменений связанных AI-объектов инициативы.
     */
    @GetMapping("/{initiativeId}/meta-ai-objects/history")
    @PlatformAudit(
        dataType = GetIiAgentHistory,
        actionType = Get,
        value = "Получение истории по связанным ИИ-агентам"
    )
    @PreAuthorize("hasAnyAuthority('TRANSFORMATION_OFFICE', 'CMS_ADMIN', 'PROJECT_OFFICE')")
    @Operation(summary = "Получение истории изменений связанных ИИ-агентов и LLM-решений")
    open fun getMetaAiObjectsHistory(
        @PathVariable initiativeId: Long,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): Page<MetaAiObjectHistoryResponse> {
        return metaAiObjectsHistoryService.getHistory(initiativeId, page, size)
    }

    /**
 * Получение истории изменений связанных AI-объектов инициативы.
 */
@Service
class MetaAiObjectsHistoryService(
    private val aiAgentRepository: AIAgentRepository,
    private val historyRepository: InitiativeAiObjectHistoryRepository,
    private val userAccountService: UserAccountService,
    private val messageProvider: MessageProvider
) {

    /**
     * Возвращает историю изменений связанных AI-объектов инициативы.
     *
     * История сортируется по createdAt DESC, id DESC.
     * ФИО редакторов получаются одним запросом в prm-auth.
     *
     * @param initiativeId идентификатор инициативы
     * @param page номер страницы, начиная с 0
     * @param size количество элементов на странице, от 1 до 100
     * @return страница истории изменений
     */
    fun getHistory(initiativeId: Long, page: Int, size: Int): Page<MetaAiObjectHistoryResponse> {
        if (initiativeId <= 0 || page < 0 || size !in 1..100) {
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

        val pageable = PageRequest.of(
            page,
            size,
            Sort.by(
                Sort.Order.desc("createdAt"),
                Sort.Order.desc("id")
            )
        )

        val historyPage = historyRepository.findByInitiativeId(initiativeId, pageable)

        val userIds = historyPage.content
            .mapNotNull { it.createdBy?.toLongOrNull() }
            .toSet()

        val usersById = userAccountService.getUsersByIds(userIds)

        return historyPage.map { history ->
            val userId = history.createdBy?.toLongOrNull()

            val editor = userId
                ?.let { usersById[it] }
                ?.let { userAccountService.getFullName(it) }
                ?: history.createdByName?.takeIf { it.isNotBlank() }
                ?: history.createdBy
                ?: ""

            MetaAiObjectHistoryResponse(
                id = history.id,
                eventAt = requireNotNull(history.createdAt) {
                    "Не указана дата события истории ${history.id}"
                },
                eventType = history.eventType,
                changedObjectsCount = history.changedObjectsCount,
                message = buildMessage(history),
                editor = editor
            )
        }
    }

    /**
     * Формирует пользовательское описание события истории.
     */
    private fun buildMessage(history: InitiativeAiObjectHistoryEntity): String {
        return when (history.eventType) {
            InitiativeAiObjectEventType.FILE_IMPORT ->
                "Получены новые сведения по ${history.changedObjectsCount} объектам инициативы"
        }
    }
}

/**
     * Возвращает историю изменений указанной инициативы.
     */
    @Query("""
        SELECT history
        FROM InitiativeAiObjectHistoryEntity history
        WHERE history.initiative.id = :initiativeId
    """)
    fun findByInitiativeId(
        @Param("initiativeId") initiativeId: Long,
        pageable: Pageable
    ): Page<InitiativeAiObjectHistoryEntity>


/**
 * Запись истории изменений связанных AI-объектов инициативы.
 */
@Schema(description = "История изменений связанных ИИ-агентов и LLM-решений")
data class MetaAiObjectHistoryResponse(

    @field:Schema(description = "ID записи истории")
    val id: Long,

    @field:Schema(description = "Дата и время события")
    val eventAt: LocalDateTime,

    @field:Schema(description = "Тип события")
    val eventType: InitiativeAiObjectEventType,

    @field:Schema(description = "Количество уникальных изменённых объектов")
    val changedObjectsCount: Int,

    @field:Schema(description = "Описание события")
    val message: String,

    @field:Schema(description = "ФИО пользователя, выполнившего изменения")
    val editor: String
)
    
```
