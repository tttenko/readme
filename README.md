```java

/**
 * Создаёт контакты новой инициативы на основании данных Jira.
 *
 * Правила создания:
 * - assignee сохраняется как developer;
 * - reporter сохраняется как customer;
 * - если reporter отсутствует, для customer используется customfield_29202;
 * - существующий контакт переиспользуется по email;
 * - отсутствующий контакт создаётся без отправки приглашения.
 *
 * Поиск существующих контактов выполняется одним запросом
 * для всех email текущей инициативы.
 */
@Component
class JiraInitiativeContactCreator(
    private val contactRepository: ContactRepository,
    private val agentContactRepository: AgentContactRepository,
) {

    companion object {
        private const val DEVELOPER_CONTACT_TYPE = "developer"
        private const val CUSTOMER_CONTACT_TYPE = "customer"

        private val log by logger()
    }

    /**
     * Создаёт contact и agent_contact для новой инициативы.
     *
     * @param agent созданная инициатива Пульта
     * @param issue исходная Jira-инициатива
     */
    fun createContacts(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
    ) {
        val contactCandidates = resolveContactCandidates(issue)

        if (contactCandidates.isEmpty()) {
            log.debug(
                "No contacts found for Jira initiative: jiraKey={}, agentId={}",
                issue.key,
                agent.id,
            )
            return
        }

        val contactEmails = contactCandidates
            .map { contactCandidate -> contactCandidate.email }
            .distinct()

        val contactsByEmail = contactRepository
            .findAllByEmailIn(contactEmails)
            .mapNotNull { contact ->
                contact.email?.let { email ->
                    email to contact
                }
            }
            .toMap()
            .toMutableMap()

        val contactsToCreate = contactCandidates
            .asSequence()
            .filter { contactCandidate ->
                contactCandidate.email !in contactsByEmail
            }
            .distinctBy { contactCandidate ->
                contactCandidate.email
            }
            .map { contactCandidate ->
                ContactEntity(
                    fio = contactCandidate.displayName,
                    email = contactCandidate.email,
                    invited = null,
                )
            }
            .toList()

        if (contactsToCreate.isNotEmpty()) {
            val createdContacts = contactRepository.saveAll(contactsToCreate)

            createdContacts.forEach { createdContact ->
                createdContact.email?.let { email ->
                    contactsByEmail[email] = createdContact
                }
            }

            log.debug(
                "Created new contacts from Jira: jiraKey={}, agentId={}, createdContacts={}",
                issue.key,
                agent.id,
                createdContacts.size,
            )
        }

        val agentContacts = contactCandidates.mapNotNull { contactCandidate ->
            val contact = contactsByEmail[contactCandidate.email]

            if (contact == null) {
                log.warn(
                    "Cannot create agent contact because contact was not resolved: jiraKey={}, agentId={}, contactType={}",
                    issue.key,
                    agent.id,
                    contactCandidate.type,
                )
                return@mapNotNull null
            }

            AgentContactEntity(
                agent = agent,
                type = contactCandidate.type,
                contact = contact,
                userId = null,
            )
        }

        if (agentContacts.isEmpty()) {
            return
        }

        agentContactRepository.saveAll(agentContacts)

        log.debug(
            "Created agent contact relations: jiraKey={}, agentId={}, contactTypes={}",
            issue.key,
            agent.id,
            agentContacts.mapNotNull { agentContact -> agentContact.type },
        )
    }

    /**
     * Извлекает контакты developer/customer из Jira.
     *
     * Для customer reporter имеет приоритет.
     * customfield_29202 используется только при отсутствии reporter.
     */
    private fun resolveContactCandidates(
        issue: SearchIssueDto,
    ): List<JiraContactCandidate> {
        val fields = issue.fields ?: return emptyList()

        val developer = fields.assignee?.let { assignee ->
            createContactCandidate(
                jiraKey = issue.key,
                type = DEVELOPER_CONTACT_TYPE,
                email = assignee.emailAddress,
                displayName = assignee.displayName,
            )
        }

        val customerSource = fields.reporter
            ?: fields.customfield_29202

        val customer = customerSource?.let { customerContact ->
            createContactCandidate(
                jiraKey = issue.key,
                type = CUSTOMER_CONTACT_TYPE,
                email = customerContact.emailAddress,
                displayName = customerContact.displayName,
            )
        }

        return listOfNotNull(
            developer,
            customer,
        )
    }

    /**
     * Создаёт внутреннее представление Jira-контакта.
     *
     * Контакт без email не создаётся, поскольку email используется
     * как идентификатор при поиске существующего контакта.
     */
    private fun createContactCandidate(
        jiraKey: String?,
        type: String,
        email: String?,
        displayName: String?,
    ): JiraContactCandidate? {
        val normalizedEmail = email
            ?.trim()
            ?.takeIf(String::isNotBlank)

        if (normalizedEmail == null) {
            log.warn(
                "Cannot resolve Jira contact because email is empty: jiraKey={}, contactType={}",
                jiraKey,
                type,
            )
            return null
        }

        return JiraContactCandidate(
            type = type,
            email = normalizedEmail,
            displayName = displayName,
        )
    }

    /**
     * Внутреннее представление контакта,
     * необходимое только для Jira-import.
     */
    private data class JiraContactCandidate(
        val type: String,
        val email: String,
        val displayName: String?,
    )
}

/**
 * Создаёт связи новой инициативы с энейблерами,
 * выбранными в Jira.
 *
 * Сопоставление выполняется по нормализованному имени:
 * значение приводится к нижнему регистру и очищается от пробелов.
 *
 * Справочник энейблеров заранее загружен один раз
 * на запуск scheduler.
 */
@Component
class JiraInitiativeEnablerCreator(
    private val enablerRepository: EnablerRepository,
    private val enablerNameNormalizer: EnablerNameNormalizer,
) {

    private val log by logger()

    /**
     * Создаёт agent_enabler для всех Jira-элементов,
     * у которых checked=true и найден соответствующий энейблер.
     *
     * @param agent созданная инициатива Пульта
     * @param issue исходная Jira-инициатива
     * @param referenceData справочники текущего запуска scheduler
     */
    fun createEnablers(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
    ) {
        val selectedEnablers = issue.fields
            ?.customfield_15903
            .orEmpty()
            .filter { jiraEnabler ->
                jiraEnabler.checked == true
            }

        if (selectedEnablers.isEmpty()) {
            log.debug(
                "No selected enablers found for Jira initiative: jiraKey={}, agentId={}",
                issue.key,
                agent.id,
            )
            return
        }

        val resolvedEnablerIds = selectedEnablers
            .mapNotNull { jiraEnabler ->
                val normalizedName = enablerNameNormalizer.normalize(
                    jiraEnabler.name
                )

                if (normalizedName == null) {
                    log.warn(
                        "Cannot resolve Jira enabler because name is empty: jiraKey={}, agentId={}",
                        issue.key,
                        agent.id,
                    )
                    return@mapNotNull null
                }

                val enabler = referenceData
                    .enablersByNormalizedName[normalizedName]

                if (enabler == null) {
                    log.warn(
                        "Jira enabler was not found in Pult dictionary: jiraKey={}, agentId={}, enablerName={}, normalizedName={}",
                        issue.key,
                        agent.id,
                        jiraEnabler.name,
                        normalizedName,
                    )

                    return@mapNotNull null
                }

                enabler.id
            }
            .distinct()

        if (resolvedEnablerIds.isEmpty()) {
            log.debug(
                "No Jira enablers were resolved in Pult dictionary: jiraKey={}, agentId={}",
                issue.key,
                agent.id,
            )
            return
        }

        enablerRepository.addAllToAgent(
            agentId = agent.id,
            enablerIds = resolvedEnablerIds,
        )

        log.debug(
            "Created agent enabler relations: jiraKey={}, agentId={}, enablerIds={}",
            issue.key,
            agent.id,
            resolvedEnablerIds,
        )
    }
}



AIAgentQualityGateRepository
/**
 * Создаёт начальные вехи новой инициативы в состоянии unchecked.
 *
 * Вставка выполняется одним запросом для всех переданных кодов.
 * Повторное создание уже существующих связей игнорируется.
 *
 * disabled=false и disabled=null считаются активными значениями.
 *
 * @param agentId идентификатор инициативы
 * @param qualityGateCodes коды активных quality gates
 * @param createdAt дата и время создания связей
 * @return количество созданных записей
 */
@Transactional
@Modifying(flushAutomatically = true)
@Query(
    value = """
        insert into agent_quality_gate (
            ai_agent_id,
            quality_gate_code,
            state,
            created
        )
        select
            :agentId,
            quality_gate.code,
            'unchecked',
            :createdAt
        from quality_gate
        where quality_gate.code in (:qualityGateCodes)
          and quality_gate.type = 'quality_gate'
          and quality_gate.disabled is not true
        on conflict (ai_agent_id, quality_gate_code)
        do nothing
    """,
    nativeQuery = true,
)
fun insertUncheckedForAgent(
    @Param("agentId")
    agentId: Long,

    @Param("qualityGateCodes")
    qualityGateCodes: Collection<String>,

    @Param("createdAt")
    createdAt: LocalDateTime,
): Int

/**
 * Создаёт начальные вехи новой Jira-инициативы.
 *
 * Для всех активных записей quality_gate типа quality_gate
 * создаётся agent_quality_gate со статусом unchecked.
 */
@Component
class JiraInitiativeQualityGateCreator(
    private val qualityGateRepository: AIAgentQualityGateRepository,
) {

    private val log by logger()

    /**
     * Инициализирует quality gates новой инициативы.
     *
     * Все связи создаются одним batch-запросом.
     *
     * @param agent созданная инициатива Пульта
     * @param referenceData справочники текущего запуска scheduler
     * @param currentDateTime единое время создания связанных данных
     */
    fun createQualityGates(
        agent: AIAgentEntity,
        referenceData: JiraImportReferenceData,
        currentDateTime: LocalDateTime,
    ) {
        val qualityGateCodes = referenceData.qualityGates
            .asSequence()
            .filter { qualityGate ->
                qualityGate.type == QualityGateType.quality_gate &&
                    qualityGate.disabled != true
            }
            .mapNotNull { qualityGate ->
                qualityGate.code
            }
            .distinct()
            .toList()

        if (qualityGateCodes.isEmpty()) {
            log.warn(
                "No active quality gates found while creating Jira initiative: agentId={}",
                agent.id,
            )
            return
        }

        val createdQualityGates = qualityGateRepository
            .insertUncheckedForAgent(
                agentId = agent.id,
                qualityGateCodes = qualityGateCodes,
                createdAt = currentDateTime,
            )

        log.debug(
            "Initialized quality gates for Jira initiative: agentId={}, requested={}, created={}",
            agent.id,
            qualityGateCodes.size,
            createdQualityGates,
        )
    }
}

/**
 * Данные GIGAUSAGE issue, связанного с Jira-инициативой.
 */
data class JiraGigaUsageIssueData(
    val jiraId: String?,
    val jiraKey: String,
)

/**
 * Находит связанные с инициативой GIGAUSAGE issues.
 *
 * Проверяются оба направления Jira issue link:
 * inwardIssue и outwardIssue.
 *
 * Если GIGAUSAGE-связей несколько, возвращаются все уникальные issues.
 */
@Component
class JiraGigaUsageIssueResolver {

    companion object {
        private const val GIGAUSAGE_KEY_PART = "GIGAUSAGE"

        private val log by logger()
    }

    /**
     * Возвращает все GIGAUSAGE issues, связанные с Jira-инициативой.
     *
     * Дубликаты одной и той же связи исключаются по Jira key.
     *
     * @param issue исходная Jira-инициатива
     * @return список найденных GIGAUSAGE issues
     */
    fun resolveGigaUsageIssues(
        issue: SearchIssueDto,
    ): List<JiraGigaUsageIssueData> {
        val gigaUsageIssues = issue.fields
            ?.issuelinks
            .orEmpty()
            .asSequence()
            .flatMap { issueLink ->
                sequenceOf(
                    issueLink.outwardIssue,
                    issueLink.inwardIssue,
                )
            }
            .filterNotNull()
            .mapNotNull { linkedIssue ->
                val jiraKey = linkedIssue.key
                    ?.trim()
                    ?.takeIf { key ->
                        key.contains(
                            other = GIGAUSAGE_KEY_PART,
                            ignoreCase = true,
                        )
                    }
                    ?: return@mapNotNull null

                JiraGigaUsageIssueData(
                    jiraId = linkedIssue.id,
                    jiraKey = jiraKey,
                )
            }
            .distinctBy { gigaUsageIssue ->
                gigaUsageIssue.jiraKey.uppercase()
            }
            .toList()

        if (gigaUsageIssues.isEmpty()) {
            log.debug(
                "No GIGAUSAGE issues found for Jira initiative: jiraKey={}",
                issue.key,
            )

            return emptyList()
        }

        log.debug(
            "Resolved GIGAUSAGE issues for Jira initiative: jiraKey={}, gigaUsageCount={}, gigaUsageKeys={}",
            issue.key,
            gigaUsageIssues.size,
            gigaUsageIssues.map { gigaUsageIssue ->
                gigaUsageIssue.jiraKey
            },
        )

        return gigaUsageIssues
    }
}

/**
 * Создаёт связи новой инициативы с Jira issues.
 *
 * В рамках третьего этапа создаются:
 * - jira_issue основной CROSSGOAL initiative;
 * - jira_issue для каждой найденной GIGAUSAGE-связи.
 *
 * Monitoring epic и его подзадачи данным компонентом
 * намеренно не обрабатываются — это следующий этап FR1.
 */
@Component
class JiraInitiativeIssueRelationCreator(
    private val jiraIssueRepository: JiraIssueRepository,
    private val jiraService: JiraService,
    private val gigaUsageIssueResolver: JiraGigaUsageIssueResolver,
) {

    companion object {
        private const val CROSSGOAL_PROJECT = "crossgoal"
        private const val GIGAUSAGE_PROJECT = "gigausage"

        private val log by logger()
    }

    /**
     * Создаёт связи текущей инициативы с Jira.
     *
     * Основная CROSSGOAL-связь создаётся всегда.
     * Все найденные GIGAUSAGE-связи создаются дополнительно.
     *
     * @param agent созданная инициатива Пульта
     * @param issue исходная Jira-инициатива
     * @param currentDateTime единое время создания связанных данных
     */
    fun createIssueRelations(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
        currentDateTime: LocalDateTime,
    ) {
        val jiraKey = requireNotNull(
            issue.key?.takeIf(String::isNotBlank)
        ) {
            "Jira initiative key must not be empty"
        }

        val jiraIssues = mutableListOf(
            createCrossgoalInitiativeIssue(
                agent = agent,
                issue = issue,
                jiraKey = jiraKey,
                currentDateTime = currentDateTime,
            )
        )

        val gigaUsageIssues = gigaUsageIssueResolver
            .resolveGigaUsageIssues(issue)

        jiraIssues += gigaUsageIssues.map { gigaUsageIssue ->
            createGigaUsageIssue(
                agent = agent,
                gigaUsageIssue = gigaUsageIssue,
                currentDateTime = currentDateTime,
            )
        }

        jiraIssueRepository.saveAll(jiraIssues)

        log.debug(
            "Created Jira issue relations: jiraKey={}, agentId={}, crossgoalCount=1, gigaUsageCount={}, totalCount={}",
            jiraKey,
            agent.id,
            gigaUsageIssues.size,
            jiraIssues.size,
        )
    }

    /**
     * Создаёт связь с основной CROSSGOAL initiative.
     */
    private fun createCrossgoalInitiativeIssue(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
        jiraKey: String,
        currentDateTime: LocalDateTime,
    ): JiraIssueEntity {
        return JiraIssueEntity(
            agent = agent,
            project = CROSSGOAL_PROJECT,
            type = JiraIssueType.initiative.name,
            jiraId = issue.id,
            jiraKey = jiraKey,
            jiraUrl = jiraService.getJiraSigmaUrl() + jiraKey,
        ).apply {
            created = currentDateTime
        }
    }

    /**
     * Создаёт связь с одной GIGAUSAGE initiative.
     */
    private fun createGigaUsageIssue(
        agent: AIAgentEntity,
        gigaUsageIssue: JiraGigaUsageIssueData,
        currentDateTime: LocalDateTime,
    ): JiraIssueEntity {
        return JiraIssueEntity(
            agent = agent,
            project = GIGAUSAGE_PROJECT,
            type = JiraIssueType.initiative.name,
            jiraId = gigaUsageIssue.jiraId,
            jiraKey = gigaUsageIssue.jiraKey,
            jiraUrl = jiraService.getJiraSigmaUrl() + gigaUsageIssue.jiraKey,
        ).apply {
            created = currentDateTime
        }
    }
}

/**
 * Создаёт новую инициативу Пульта на основании данных Jira.
 *
 * В рамках одной транзакции текущая реализация FR1:
 * 1. создаёт AIAgentEntity;
 * 2. создаёт связи со стратегиями;
 * 3. сохраняет задействованные ресурсы;
 * 4. создаёт developer/customer контакты;
 * 5. создаёт связи с энейблерами;
 * 6. инициализирует quality gates в состоянии unchecked;
 * 7. создаёт связь с основной CROSSGOAL initiative;
 * 8. создаёт связи со всеми найденными GIGAUSAGE issues.
 *
 * Monitoring epic, связанные задачи и последующий расчёт статуса
 * будут обрабатываться на следующих этапах FR1.
 *
 * Каждая Jira-инициатива обрабатывается в отдельной транзакции,
 * поэтому ошибка одной инициативы не откатывает обработку остальных.
 */
@Service
class JiraNewInitiativeCreator(
    private val agentRepository: AIAgentRepository,
    private val agentStrategyRepository: AgentStrategyRepository,
    private val involvedResourceRepository: InvolvedResourceRepository,

    private val organizationResolver: JiraInitiativeOrganizationResolver,
    private val initiativeTypeResolver: JiraInitiativeTypeResolver,
    private val numericValueParser: JiraNumericValueParser,
    private val strategyResolver: JiraStrategyResolver,
    private val involvedResourceResolver: JiraInvolvedResourceResolver,

    private val contactCreator: JiraInitiativeContactCreator,
    private val enablerCreator: JiraInitiativeEnablerCreator,
    private val qualityGateCreator: JiraInitiativeQualityGateCreator,
    private val issueRelationCreator: JiraInitiativeIssueRelationCreator,
) {

    companion object {
        private const val ANALYSIS_STATUS_CODE = "analysis"

        private const val IMPORT_STATUS_BLOCKED = "blocked"
        private const val JIRA_FROM_STATUS_IN_PROGRESS = "inProgress"
        private const val JIRA_LINK_DONE = "done"

        private const val MAX_AGENT_NAME_LENGTH = 255
        private const val MAX_AGENT_DESCRIPTION_LENGTH = 2000

        private val log by logger()
    }

    /**
     * Создаёт новую инициативу и связанные с ней данные
     * текущих этапов FR1.
     *
     * Все операции выполняются атомарно в отдельной транзакции.
     *
     * @param issue исходная Jira-инициатива
     * @param referenceData справочники текущего запуска scheduler
     */
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        rollbackFor = [Exception::class],
    )
    fun createInitiativeFromJira(
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
    ) {
        val jiraKey = requireNotNull(
            issue.key?.takeIf(String::isNotBlank)
        ) {
            "Jira initiative key must not be empty"
        }

        log.debug(
            "Started creating new Jira initiative in Pult: jiraKey={}",
            jiraKey,
        )

        val analysisStatus = requireNotNull(
            referenceData.statusesByCode[ANALYSIS_STATUS_CODE]
        ) {
            "Active status '$ANALYSIS_STATUS_CODE' is not configured"
        }

        val organization = organizationResolver.resolveOrganization(
            initiatorUnits = issue.fields?.customfield_30000,
            executorUnits = issue.fields?.customfield_30001,
            referenceData = referenceData,
        )

        if (
            organization.block == null &&
            organization.division == null
        ) {
            log.debug(
                "Block and division were not resolved for new Jira initiative: jiraKey={}, customfield_30000={}, customfield_30001={}",
                jiraKey,
                issue.fields?.customfield_30000,
                issue.fields?.customfield_30001,
            )
        }

        val initiativeType = initiativeTypeResolver.resolveInitiativeType(
            labels = issue.fields?.labels,
            initiativeTypesByCode = referenceData.initiativeTypesByCode,
        )

        val optimizationEffect = parseEffect(
            jiraKey = jiraKey,
            jiraFieldName = "customfield_34300",
            jiraFieldValue = issue.fields?.customfield_34300,
        )

        val revenueEffect = parseEffect(
            jiraKey = jiraKey,
            jiraFieldName = "customfield_30401",
            jiraFieldValue = issue.fields?.customfield_30401,
        )

        val currentDateTime = LocalDateTime.now()

        val agent = AIAgentEntity(
            agentId = jiraKey,

            agentName = truncate(
                jiraKey = jiraKey,
                jiraFieldName = "summary",
                value = issue.fields?.summary,
                maxLength = MAX_AGENT_NAME_LENGTH,
            ),

            agentDescription = truncate(
                jiraKey = jiraKey,
                jiraFieldName = "description",
                value = issue.fields?.description,
                maxLength = MAX_AGENT_DESCRIPTION_LENGTH,
            ),

            agentJiraUrl = jiraKey,

            agentEffectOptimization = optimizationEffect,
            agentEffectRevenue = revenueEffect,

            initiativeType = initiativeType,

            block = organization.block,
            division = organization.division,

            agentStatus = analysisStatus,

            importStatus = IMPORT_STATUS_BLOCKED,
            jiraFromStatus = JIRA_FROM_STATUS_IN_PROGRESS,
        ).apply {
            created = currentDateTime
        }

        val savedAgent = agentRepository.save(agent)

        log.debug(
            "Created ai_agent from Jira: jiraKey={}, agentId={}, block={}, division={}, initiativeType={}",
            jiraKey,
            savedAgent.id,
            savedAgent.block?.code,
            savedAgent.division?.code,
            savedAgent.initiativeType?.code,
        )

        createStrategies(
            agent = savedAgent,
            issue = issue,
            referenceData = referenceData,
        )

        createInvolvedResources(
            agent = savedAgent,
            issue = issue,
            jiraKey = jiraKey,
            currentDateTime = currentDateTime,
        )

        contactCreator.createContacts(
            agent = savedAgent,
            issue = issue,
        )

        enablerCreator.createEnablers(
            agent = savedAgent,
            issue = issue,
            referenceData = referenceData,
        )

        qualityGateCreator.createQualityGates(
            agent = savedAgent,
            referenceData = referenceData,
            currentDateTime = currentDateTime,
        )

        issueRelationCreator.createIssueRelations(
            agent = savedAgent,
            issue = issue,
            currentDateTime = currentDateTime,
        )

        log.info(
            "Successfully created new Jira initiative in Pult: jiraKey={}, agentId={}",
            jiraKey,
            savedAgent.id,
        )
    }

    /**
     * Создаёт связи новой инициативы со стратегиями.
     *
     * Для найденных Jira-связей создаётся agent_strategy
     * со значением jira_link=done.
     */
    private fun createStrategies(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
        referenceData: JiraImportReferenceData,
    ) {
        val strategies = strategyResolver.resolveStrategies(
            issue = issue,
            strategiesByJiraKey = referenceData.strategiesByJiraKey,
        )

        if (strategies.isEmpty()) {
            log.debug(
                "No strategies found for Jira initiative: jiraKey={}",
                issue.key,
            )
            return
        }

        val agentStrategies = strategies.map { strategy ->
            AgentStrategyEntity(
                agent = agent,
                strategy = strategy,
                jiraLink = JIRA_LINK_DONE,
            )
        }

        agentStrategyRepository.saveAll(agentStrategies)

        log.debug(
            "Created agent strategies: jiraKey={}, agentId={}, strategyIds={}",
            issue.key,
            agent.id,
            strategies.map { strategy ->
                strategy.id
            },
        )
    }

    /**
     * Создаёт involved_resource для новой инициативы.
     *
     * Каждое успешно распарсенное Jira-поле ресурсов
     * сохраняется отдельной записью.
     */
    private fun createInvolvedResources(
        agent: AIAgentEntity,
        issue: SearchIssueDto,
        jiraKey: String,
        currentDateTime: LocalDateTime,
    ) {
        val resources = involvedResourceResolver.resolveInvolvedResources(
            jiraKey = jiraKey,
            issue = issue,
        )

        if (resources.isEmpty()) {
            log.debug(
                "No involved resources found for Jira initiative: jiraKey={}",
                jiraKey,
            )
            return
        }

        val involvedResourceEntities = resources.map { resource ->
            InvolvedResourceEntity().apply {
                id = InvolvedResourceEmbeddedId(
                    aiAgentId = agent.id,
                    source = resource.source,
                    type = resource.type,
                )

                value = resource.value
                timeAllocated = null
                created = currentDateTime
                updated = currentDateTime
                aiAgent = agent
            }
        }

        involvedResourceRepository.saveAll(involvedResourceEntities)

        log.debug(
            "Created involved resources: jiraKey={}, agentId={}, resources={}",
            jiraKey,
            agent.id,
            resources.map { resource ->
                "${resource.source}/${resource.type}=${resource.value}"
            },
        )
    }

    /**
     * Извлекает числовое значение эффекта из Jira-поля.
     *
     * Некорректное непустое значение не блокирует создание инициативы,
     * но фиксируется в логах.
     */
    private fun parseEffect(
        jiraKey: String,
        jiraFieldName: String,
        jiraFieldValue: String?,
    ): BigDecimal? {
        if (jiraFieldValue.isNullOrBlank()) {
            return null
        }

        val numericValue = numericValueParser.parseFirst(jiraFieldValue)

        if (numericValue == null) {
            log.warn(
                "Cannot parse initiative effect from Jira: jiraKey={}, field={}, value={}",
                jiraKey,
                jiraFieldName,
                jiraFieldValue,
            )
        }

        return numericValue
    }

    /**
     * Ограничивает строковое значение максимально допустимой длиной.
     *
     * Если значение превышает допустимый размер,
     * оно обрезается и операция фиксируется в DEBUG-логе.
     */
    private fun truncate(
        jiraKey: String,
        jiraFieldName: String,
        value: String?,
        maxLength: Int,
    ): String? {
        if (
            value == null ||
            value.length <= maxLength
        ) {
            return value
        }

        log.debug(
            "Jira field was truncated while creating initiative: jiraKey={}, field={}, originalLength={}, maxLength={}",
            jiraKey,
            jiraFieldName,
            value.length,
            maxLength,
        )

        return value.take(maxLength)
    }
}

```
