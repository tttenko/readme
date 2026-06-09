```java

@Entity
@Table(name = "jira_change")
open class JiraChangeEntity(

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(
        name = "agent_id",
        referencedColumnName = "id",
        nullable = false,
    )
    open var agent: AIAgentEntity? = null,

    @Column(
        name = "change_type",
        length = 50,
        nullable = false,
    )
    open var changeType: String? = null,

    @Type(JsonType::class)
    @Column(
        name = "payload",
        columnDefinition = "jsonb",
        nullable = false,
    )
    open var payload: JsonNode? = null,

    @Column(
        name = "created",
        nullable = false,
    )
    open var created: LocalDateTime = LocalDateTime.now(),

) : BasicLongEntity()

@Repository
interface JiraChangeRepository :
    JpaRepository<JiraChangeEntity, Long>

data class JiraPlannedDateDto(
    val status: String,
    val plannedDate: Instant,
)

data class JiraPlannedDatePayload(
    val statusSla: List<JiraPlannedDateDto>,
)

@Component
open class JiraChangeUpdater(
    private val jiraIssueRepository: JiraIssueRepository,
    private val jiraChangeRepository: JiraChangeRepository,
    private val objectMapper: ObjectMapper,
) {

    open fun update(
        agent: AIAgentEntity,
        request: Map<String, Any?>,
        changedStatusSla: List<StatusSlaDto>,
        userId: Long,
    ) {
        val agentId = agent.id

        /*
         * Проверяем, привязана ли инициатива к CROSSGOAL.
         */
        val hasCrossgoalInitiative =
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = agentId,
                type = INITIATIVE_ISSUE_TYPE,
                project = CROSSGOAL_PROJECT,
            )

        /*
         * Если связи с CROSSGOAL нет,
         * синхронизация не требуется.
         */
        if (!hasCrossgoalInitiative) {
            return
        }

        val currentDateTime = LocalDateTime.now()

        val jiraChangesToSave =
            mutableListOf<JiraChangeEntity>()

        createInitiativeChange(
            agent = agent,
            request = request,
            created = currentDateTime,
        )?.let { initiativeChange ->
            jiraChangesToSave.add(initiativeChange)
        }

        createPlannedDateChange(
            agent = agent,
            changedStatusSla = changedStatusSla,
            created = currentDateTime,
        )?.let { plannedDateChange ->
            jiraChangesToSave.add(plannedDateChange)
        }

        /*
         * Ни одного изменения для JIRA не найдено.
         */
        if (jiraChangesToSave.isEmpty()) {
            return
        }

        jiraChangeRepository.saveAll(
            jiraChangesToSave
        )

        /*
         * Если создана хотя бы одна запись jira_change,
         * выставляем pendingUpdate.
         */
        agent.jiraStatus = PENDING_UPDATE_STATUS
        agent.updated = currentDateTime
        agent.updatedBy = userId
    }

    private fun createInitiativeChange(
        agent: AIAgentEntity,
        request: Map<String, Any?>,
        created: LocalDateTime,
    ): JiraChangeEntity? {

        /*
         * В payload помещаются только поля,
         * перечисленные в документации.
         *
         * filterKeys сохраняет и поля со значением null,
         * если ключ присутствовал в PATCH-запросе.
         */
        val initiativePayload = request
            .filterKeys { fieldName ->
                fieldName in INITIATIVE_SYNC_FIELDS
            }

        if (initiativePayload.isEmpty()) {
            return null
        }

        return JiraChangeEntity(
            agent = agent,
            changeType = INITIATIVE_CHANGE_TYPE,
            payload = objectMapper.valueToTree(
                initiativePayload
            ),
            created = created,
        )
    }

    private fun createPlannedDateChange(
        agent: AIAgentEntity,
        changedStatusSla: List<StatusSlaDto>,
        created: LocalDateTime,
    ): JiraChangeEntity? {

        /*
         * StatusSlaUpdater уже вернул только дельту.
         */
        if (changedStatusSla.isEmpty()) {
            return null
        }

        val changedPlannedDates =
            changedStatusSla.map { statusSla ->

                val status =
                    requireNotNull(statusSla.status)

                val plannedDate =
                    requireNotNull(statusSla.plannedDate)

                JiraPlannedDateDto(
                    status = status,
                    plannedDate = plannedDate
                        .atStartOfDay()
                        .toInstant(ZoneOffset.UTC),
                )
            }

        val plannedDatePayload =
            JiraPlannedDatePayload(
                statusSla = changedPlannedDates,
            )

        return JiraChangeEntity(
            agent = agent,
            changeType = PLANNED_DATE_CHANGE_TYPE,
            payload = objectMapper.valueToTree(
                plannedDatePayload
            ),
            created = created,
        )
    }

    private companion object {
        const val CROSSGOAL_PROJECT = "crossgoal"
        const val INITIATIVE_ISSUE_TYPE = "initiative"

        const val INITIATIVE_CHANGE_TYPE = "initiative"
        const val PLANNED_DATE_CHANGE_TYPE = "plannedDate"

        const val PENDING_UPDATE_STATUS = "pendingUpdate"

        val INITIATIVE_SYNC_FIELDS = setOf(
            "agentName",
            "agentDescription",
            "agentInitiativeType",
            "block",
            "division",
            "strategies",
            "processes",
            "enablers",
            "involvedResources",
            "agentEffectOptimization",
            "agentEffectRevenue",
        )
    }
}

@Component
open class JiraChangeUpdater(
    private val jiraIssueRepository: JiraIssueRepository,
    private val jiraChangeRepository: JiraChangeRepository,
    private val objectMapper: ObjectMapper,
) {

    open fun update(
        agent: AIAgentEntity,
        request: Map<String, Any?>,
        changedStatusSla: List<StatusSlaDto>,
        userId: Long,
    ) {
        val agentId = agent.id

        /*
         * Проверяем, привязана ли инициатива к CROSSGOAL.
         */
        val hasCrossgoalInitiative =
            jiraIssueRepository.existsByAgentIdAndTypeAndProject(
                agentId = agentId,
                type = INITIATIVE_ISSUE_TYPE,
                project = CROSSGOAL_PROJECT,
            )

        /*
         * Если связи с CROSSGOAL нет,
         * синхронизация не требуется.
         */
        if (!hasCrossgoalInitiative) {
            return
        }

        val currentDateTime = LocalDateTime.now()

        val jiraChangesToSave =
            mutableListOf<JiraChangeEntity>()

        createInitiativeChange(
            agent = agent,
            request = request,
            created = currentDateTime,
        )?.let { initiativeChange ->
            jiraChangesToSave.add(initiativeChange)
        }

        createPlannedDateChange(
            agent = agent,
            changedStatusSla = changedStatusSla,
            created = currentDateTime,
        )?.let { plannedDateChange ->
            jiraChangesToSave.add(plannedDateChange)
        }

        /*
         * Ни одного изменения для JIRA не найдено.
         */
        if (jiraChangesToSave.isEmpty()) {
            return
        }

        jiraChangeRepository.saveAll(
            jiraChangesToSave
        )

        /*
         * Если создана хотя бы одна запись jira_change,
         * выставляем pendingUpdate.
         */
        agent.jiraStatus = PENDING_UPDATE_STATUS
        agent.updated = currentDateTime
        agent.updatedBy = userId
    }

    private fun createInitiativeChange(
        agent: AIAgentEntity,
        request: Map<String, Any?>,
        created: LocalDateTime,
    ): JiraChangeEntity? {

        /*
         * В payload помещаются только поля,
         * перечисленные в документации.
         *
         * filterKeys сохраняет и поля со значением null,
         * если ключ присутствовал в PATCH-запросе.
         */
        val initiativePayload = request
            .filterKeys { fieldName ->
                fieldName in INITIATIVE_SYNC_FIELDS
            }

        if (initiativePayload.isEmpty()) {
            return null
        }

        return JiraChangeEntity(
            agent = agent,
            changeType = INITIATIVE_CHANGE_TYPE,
            payload = objectMapper.valueToTree(
                initiativePayload
            ),
            created = created,
        )
    }

    private fun createPlannedDateChange(
        agent: AIAgentEntity,
        changedStatusSla: List<StatusSlaDto>,
        created: LocalDateTime,
    ): JiraChangeEntity? {

        /*
         * StatusSlaUpdater уже вернул только дельту.
         */
        if (changedStatusSla.isEmpty()) {
            return null
        }

        val changedPlannedDates =
            changedStatusSla.map { statusSla ->

                val status =
                    requireNotNull(statusSla.status)

                val plannedDate =
                    requireNotNull(statusSla.plannedDate)

                JiraPlannedDateDto(
                    status = status,
                    plannedDate = plannedDate
                        .atStartOfDay()
                        .toInstant(ZoneOffset.UTC),
                )
            }

        val plannedDatePayload =
            JiraPlannedDatePayload(
                statusSla = changedPlannedDates,
            )

        return JiraChangeEntity(
            agent = agent,
            changeType = PLANNED_DATE_CHANGE_TYPE,
            payload = objectMapper.valueToTree(
                plannedDatePayload
            ),
            created = created,
        )
    }

    private companion object {
        const val CROSSGOAL_PROJECT = "crossgoal"
        const val INITIATIVE_ISSUE_TYPE = "initiative"

        const val INITIATIVE_CHANGE_TYPE = "initiative"
        const val PLANNED_DATE_CHANGE_TYPE = "plannedDate"

        const val PENDING_UPDATE_STATUS = "pendingUpdate"

        val INITIATIVE_SYNC_FIELDS = setOf(
            "agentName",
            "agentDescription",
            "agentInitiativeType",
            "block",
            "division",
            "strategies",
            "processes",
            "enablers",
            "involvedResources",
            "agentEffectOptimization",
            "agentEffectRevenue",
        )
    }
}


```
