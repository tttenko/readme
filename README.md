```java
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.sun.net.httpserver.HttpExchange
import com.sun.net.httpserver.HttpServer
import io.kotest.matchers.collections.shouldContainExactlyInAnyOrder
import io.kotest.matchers.nulls.shouldNotBeNull
import io.kotest.matchers.shouldBe
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import java.math.BigDecimal
import java.net.InetSocketAddress
import java.nio.charset.StandardCharsets
import java.time.LocalDateTime
import java.util.concurrent.CopyOnWriteArrayList
import java.util.concurrent.Executors

/*
 * Project imports:
 *
 * AbstractIntegrationTest
 *
 * JiraNewInitiativeImportService
 *
 * SearchIssueRequestDto
 * SearchIssueResponseDto
 * SearchIssueDto
 * SearchIssueFieldsDto
 * SearchIssueStatusDto
 * SearchIssueLinkDto
 * SearchLinkedIssueDto
 * SearchLinkedIssueFieldsDto
 * SearchIssueUserInfoDto
 * SearchIssueCheckboxOptionDto
 *
 * AIAgentEntity
 * OptionsEntity
 * StatusEntity
 * BlockEntity
 * DivisionEntity
 * InitiativeTypeEntity
 * StrategyEntity
 * EnablerEntity
 * QualityGateEntity
 *
 * QualityGateType
 * JiraIssueType
 *
 * AIAgentRepository
 * OptionsRepository
 * StatusRepository
 * BlockRepository
 * DivisionRepository
 * InitiativeTypeRepository
 * StrategyRepository
 * EnablerRepository
 * ContactRepository
 * AgentContactRepository
 * AgentStrategyRepository
 * InvolvedResourceRepository
 * JiraIssueRepository
 * AIAgentQualityGateRepository
 * AgentStatusSlaRepository
 */

/**
 * Сквозной integration test FR1.
 *
 * Проверяет взаимодействие всех этапов:
 *
 * 1. Постраничный поиск новых инициатив в Jira.
 * 2. Пропуск уже существующей инициативы.
 * 3. Создание новой ai_agent и связанных данных.
 * 4. Создание контактов, энейблеров, quality gates,
 *    CROSSGOAL/GIGAUSAGE jira_issue.
 * 5. Поиск monitoring epic и постраничное получение Task.
 * 6. Сопоставление Task с quality gates.
 * 7. Обновление agent_quality_gate, SLA и текущего статуса.
 * 8. Создание jira_issue для epic и Task.
 * 9. Завершение Jira synchronization со статусом done.
 *
 * Внутренние сервисы FR1 не мокируются.
 * Jira/prm-integr заменяется локальным HTTP stub,
 * поэтому также проверяется реальный Feign HTTP-вызов
 * и сериализация SearchIssueRequestDto/SearchIssueResponseDto.
 */
@ActiveProfiles("integration-test")
class JiraNewInitiativeImportIntegrationTest :
    AbstractIntegrationTest() {

    @Autowired
    private lateinit var jiraNewInitiativeImportService:
        JiraNewInitiativeImportService

    @Autowired
    private lateinit var agentRepository:
        AIAgentRepository

    @Autowired
    private lateinit var optionsRepository:
        OptionsRepository

    @Autowired
    private lateinit var statusRepository:
        StatusRepository

    @Autowired
    private lateinit var blockRepository:
        BlockRepository

    @Autowired
    private lateinit var divisionRepository:
        DivisionRepository

    @Autowired
    private lateinit var initiativeTypeRepository:
        InitiativeTypeRepository

    @Autowired
    private lateinit var strategyRepository:
        StrategyRepository

    @Autowired
    private lateinit var enablerRepository:
        EnablerRepository

    @Autowired
    private lateinit var qualityGateRepository:
        QualityGateRepository

    @Autowired
    private lateinit var contactRepository:
        ContactRepository

    @Autowired
    private lateinit var agentContactRepository:
        AgentContactRepository

    @Autowired
    private lateinit var agentStrategyRepository:
        AgentStrategyRepository

    @Autowired
    private lateinit var involvedResourceRepository:
        InvolvedResourceRepository

    @Autowired
    private lateinit var jiraIssueRepository:
        JiraIssueRepository

    @Autowired
    private lateinit var agentQualityGateRepository:
        AIAgentQualityGateRepository

    @Autowired
    private lateinit var agentStatusSlaRepository:
        AgentStatusSlaRepository

    @Autowired
    private lateinit var jdbcTemplate:
        JdbcTemplate

    private lateinit var analysisStatus: StatusEntity
    private lateinit var developmentStatus: StatusEntity

    private lateinit var testBlock: BlockEntity
    private lateinit var testDivision: DivisionEntity

    private lateinit var agentInitiativeType: InitiativeTypeEntity

    private lateinit var strategy: StrategyEntity
    private lateinit var enabler: EnablerEntity

    init {

        beforeTest {
            /*
             * Важно: сам test НЕ @Transactional.
             *
             * Production-код FR1 использует REQUIRES_NEW,
             * поэтому подготовленные данные должны быть
             * закоммичены и доступны этим транзакциям.
             */
            clearTables()

            jiraStub.reset()

            prepareReferenceData()
            prepareExistingInitiative()
        }

        afterTest {
            clearTables()
            jiraStub.reset()
        }

        afterSpec {
            jiraStub.stop()
        }

        "should import new Jira initiative with all related monitoring data" {

            /*
             * WHEN
             *
             * Запускаем непосредственно полный use-case FR1.
             *
             * Cron/ShedLock здесь не тестируем:
             * это инфраструктурная оболочка над данным сервисом.
             */
            jiraNewInitiativeImportService.importNewInitiatives()

            /*
             * THEN
             */
            val agent =
                agentRepository.findFirstByAgentId(NEW_INITIATIVE_KEY)

            agent.shouldNotBeNull()

            assertAgent(agent)
            assertStrategy(agent)
            assertInvolvedResources(agent)
            assertContacts(agent)
            assertEnablers(agent)
            assertQualityGates(agent)
            assertJiraIssues(agent)
            assertStatusSla(agent)

            assertExistingInitiativeWasNotDuplicated()
            assertJiraSearchRequests()
        }
    }

    /**
     * Подготавливает справочники, необходимые полному FR1.
     *
     * maxResults=1 задан намеренно:
     * и основной поиск инициатив, и monitoring Task
     * должны пройти через две страницы.
     */
    private fun prepareReferenceData() {

        optionsRepository.saveAndFlush(
            OptionsEntity().apply {
                newDepth = NEW_DEPTH
                updateDepth = 3
                maxResults = PAGE_SIZE
            }
        )

        analysisStatus =
            statusRepository.saveAndFlush(
                StatusEntity().apply {
                    name = "Концепция"
                    code = ANALYSIS_STATUS_CODE
                    ordering = 10
                    disabled = false
                }
            )

        developmentStatus =
            statusRepository.saveAndFlush(
                StatusEntity().apply {
                    name = "Разработка"
                    code = DEVELOPMENT_STATUS_CODE
                    ordering = 20
                    disabled = false
                }
            )

        /*
         * Добавляем targetSolution, чтобы reference data
         * представляла полноценный набор статусов,
         * используемый status resolver.
         *
         * В этом сценарии он не станет итоговым,
         * поскольку Jira вернёт development Task
         * в статусе InProgress.
         */
        statusRepository.saveAndFlush(
            StatusEntity().apply {
                name = "Целевое решение"
                code = TARGET_SOLUTION_STATUS_CODE
                ordering = 30
                disabled = false
            }
        )

        testBlock =
            blockRepository.saveAndFlush(
                BlockEntity().apply {
                    code = TEST_BLOCK_CODE
                    shortName = "TB"
                    name = "Test Block"
                    label = TEST_BLOCK_LABEL
                    disabled = false
                }
            )

        testDivision =
            divisionRepository.saveAndFlush(
                DivisionEntity().apply {
                    block = testBlock
                    code = TEST_DIVISION_CODE
                    shortName = "TD"
                    name = "Test Division"
                    label = TEST_DIVISION_LABEL
                    ordering = 10
                    disabled = false
                }
            )

        agentInitiativeType =
            initiativeTypeRepository.saveAndFlush(
                InitiativeTypeEntity(
                    code = AGENT_INITIATIVE_TYPE_CODE,
                    name = "AI-агент",
                    description = "AI agent",
                )
            )

        initiativeTypeRepository.saveAndFlush(
            InitiativeTypeEntity(
                code = GENERATIVE_AI_INITIATIVE_TYPE_CODE,
                name = "GenAI solution",
                description = "Generative AI solution",
            )
        )

        strategy =
            strategyRepository.saveAndFlush(
                StrategyEntity(
                    jiraIssue = STRATEGY_JIRA_KEY,
                    name = "Стратегия 2026",
                )
            )

        enabler =
            enablerRepository.saveAndFlush(
                EnablerEntity().apply {
                    name = ENABLER_DICTIONARY_NAME
                    shortDescription = "Test enabler"
                    description = "Integration test enabler"
                    disabled = false
                }
            )

        /*
         * Веха, для которой Jira вернёт завершённую Task.
         * После monitoring она должна стать checked.
         */
        qualityGateRepository.saveAndFlush(
            QualityGateEntity(
                code = COMPLETED_QUALITY_GATE_CODE,
                name = "Архитектурное согласование",
                type = QualityGateType.quality_gate,
                status = null,
                ordering = 10,
                regexp = "архитектура,согласование",
                disabled = false,
            )
        )

        /*
         * Вторая активная веха.
         *
         * Для неё Jira Task не приходит, поэтому она должна
         * остаться unchecked после всего FR1.
         */
        qualityGateRepository.saveAndFlush(
            QualityGateEntity(
                code = UNCHECKED_QUALITY_GATE_CODE,
                name = "Безопасность",
                type = QualityGateType.quality_gate,
                status = null,
                ordering = 20,
                regexp = "безопасность",
                disabled = false,
            )
        )

        /*
         * Этап инициативы.
         *
         * Jira Task этого этапа имеет status.id=3 (InProgress),
         * поэтому итоговый agent_status должен стать development.
         */
        qualityGateRepository.saveAndFlush(
            QualityGateEntity(
                code = DEVELOPMENT_STAGE_CODE,
                name = "Разработка",
                type = QualityGateType.status,
                status = developmentStatus,
                ordering = 20,
                regexp = "этап,разработка",
                disabled = false,
            )
        )
    }

    /**
     * Создаёт инициативу, которая будет возвращена
     * второй страницей Jira Search.
     *
     * FR1 должен определить её как уже существующую
     * через ai_agent.agent_jira_url и не создать дубль.
     */
    private fun prepareExistingInitiative() {
        agentRepository.saveAndFlush(
            AIAgentEntity().apply {
                agentId = EXISTING_INITIATIVE_KEY
                agentName = "Existing Jira initiative"
                agentJiraUrl = EXISTING_INITIATIVE_KEY

                block = testBlock
                division = testDivision
                initiativeType = agentInitiativeType
                agentStatus = analysisStatus

                importStatus = "blocked"
                jiraFromStatus = "done"

                disabled = false
            }
        )
    }

    /**
     * Проверяет основную запись ai_agent,
     * включая mapping Jira-полей и результат monitoring.
     */
    private fun assertAgent(
        agent: AIAgentEntity,
    ) {
        agent.agentId shouldBe NEW_INITIATIVE_KEY
        agent.agentName shouldBe NEW_INITIATIVE_SUMMARY

        /*
         * Фиксируем актуальное бизнес-правило:
         * description ограничивается 2000 символами.
         */
        agent.agentDescription shouldBe
            LONG_DESCRIPTION.take(MAX_DESCRIPTION_LENGTH)

        agent.agentDescription?.length shouldBe MAX_DESCRIPTION_LENGTH

        agent.agentJiraUrl shouldBe NEW_INITIATIVE_KEY

        agent.agentEffectOptimization
            ?.compareTo(BigDecimal("123.45")) shouldBe 0

        agent.agentEffectRevenue
            ?.compareTo(BigDecimal("678.90")) shouldBe 0

        agent.block?.id shouldBe testBlock.id
        agent.division?.id shouldBe testDivision.id

        agent.initiativeType?.code shouldBe
            AGENT_INITIATIVE_TYPE_CODE

        /*
         * Первоначально initiative создаётся в analysis,
         * но monitoring Task этапа development находится
         * в Jira status=InProgress, поэтому текущий статус
         * после обработки должен стать development.
         */
        agent.agentStatus?.code shouldBe
            DEVELOPMENT_STATUS_CODE

        agent.importStatus shouldBe "blocked"
        agent.jiraFromStatus shouldBe "done"

        agent.disabled shouldBe false

        agent.created.shouldNotBeNull()
        agent.updated.shouldNotBeNull()
        agent.jiraUpdated.shouldNotBeNull()
    }

    /**
     * Проверяет agent_strategy.
     */
    private fun assertStrategy(
        agent: AIAgentEntity,
    ) {
        val strategies =
            agentStrategyRepository.findAllByAgentId(agent.id)

        strategies.size shouldBe 1

        val agentStrategy = strategies.single()

        agentStrategy.agent?.id shouldBe agent.id
        agentStrategy.strategy?.id shouldBe strategy.id
        agentStrategy.jiraLink shouldBe "done"
    }

    /**
     * Проверяет involved_resource.
     */
    private fun assertInvolvedResources(
        agent: AIAgentEntity,
    ) {
        val resources =
            involvedResourceRepository.findAllByAiAgentId(
                agent.id
            )

        resources.size shouldBe 1

        val resource = resources.single()

        resource.id.source shouldBe "without_steerCo"
        resource.id.type shouldBe "business"

        resource.value
            ?.compareTo(BigDecimal("12.5")) shouldBe 0

        resource.timeAllocated shouldBe null

        /*
         * В актуальной модели involved_resource
         * created присутствует.
         */
        resource.created.shouldNotBeNull()
        resource.updated.shouldNotBeNull()
    }

    /**
     * Проверяет создание developer/customer.
     *
     * Оба контакта новые.
     * Поле invited должно остаться null:
     * контакты из Jira не получают приглашение.
     */
    private fun assertContacts(
        agent: AIAgentEntity,
    ) {
        val contacts = loadAgentContacts(agent.id)

        contacts.size shouldBe 2

        contacts
            .map { contact ->
                contact.type to contact.email
            }
            .shouldContainExactlyInAnyOrder(
                "developer" to DEVELOPER_EMAIL,
                "customer" to CUSTOMER_EMAIL,
            )

        val developer =
            contacts.single {
                it.type == "developer"
            }

        developer.email shouldBe DEVELOPER_EMAIL
        developer.fio shouldBe DEVELOPER_NAME
        developer.invited shouldBe null

        val customer =
            contacts.single {
                it.type == "customer"
            }

        customer.email shouldBe CUSTOMER_EMAIL
        customer.fio shouldBe CUSTOMER_NAME
        customer.invited shouldBe null

        /*
         * В Jira payload customfield_29202 тоже заполнен,
         * но reporter имеет приоритет.
         */
        contactRepository
            .findFirstByEmail(FALLBACK_CUSTOMER_EMAIL) shouldBe null
    }

    /**
     * Проверяет нормализацию enabler name и agent_enabler.
     */
    private fun assertEnablers(
        agent: AIAgentEntity,
    ) {
        val enablers =
            enablerRepository.findAllByAgentId(agent.id)

        enablers.size shouldBe 1
        enablers.single().id shouldBe enabler.id
    }

    /**
     * Проверяет начальную и monitoring-обработку
     * agent_quality_gate.
     *
     * COMPLETED -> checked
     * отсутствующая Jira Task -> unchecked
     */
    private fun assertQualityGates(
        agent: AIAgentEntity,
    ) {
        val states =
            agentQualityGateRepository
                .findAll()
                .filter { qualityGate ->
                    qualityGate.primaryKey.aiAgentId ==
                        agent.id
                }
                .associate { qualityGate ->
                    qualityGate.primaryKey.qualityGateCode to
                        qualityGate.state?.name
                }

        states shouldBe
            mapOf(
                COMPLETED_QUALITY_GATE_CODE to "checked",
                UNCHECKED_QUALITY_GATE_CODE to "unchecked",
            )
    }

    /**
     * Проверяет весь graph jira_issue:
     *
     * CROSSGOAL initiative
     * 2 x GIGAUSAGE
     * monitoring epic
     * 2 x Task
     */
    private fun assertJiraIssues(
        agent: AIAgentEntity,
    ) {

        val crossgoalInitiatives =
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = agent.id,
                type = JiraIssueType.initiative.name,
                project = "crossgoal",
            )

        crossgoalInitiatives.size shouldBe 1

        val crossgoalInitiative =
            crossgoalInitiatives.single()

        crossgoalInitiative.jiraId shouldBe
            NEW_INITIATIVE_JIRA_ID

        crossgoalInitiative.jiraKey shouldBe
            NEW_INITIATIVE_KEY

        crossgoalInitiative.jiraUrl shouldBe
            "$SIGMA_URL_PREFIX$NEW_INITIATIVE_KEY"

        val gigaUsageIssues =
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = agent.id,
                type = JiraIssueType.initiative.name,
                project = "gigausage",
            )

        gigaUsageIssues
            .mapNotNull { it.jiraKey }
            .toSet() shouldBe
            setOf(
                FIRST_GIGAUSAGE_KEY,
                SECOND_GIGAUSAGE_KEY,
            )

        /*
         * Важное актуальное правило:
         * сохраняем ВСЕ GIGAUSAGE,
         * а не элемент с максимальным Jira id.
         */
        gigaUsageIssues.size shouldBe 2

        gigaUsageIssues.forEach { issue ->
            issue.jiraUrl shouldBe
                "$SIGMA_URL_PREFIX${issue.jiraKey}"
        }

        val epics =
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = agent.id,
                type = JiraIssueType.epic.name,
                project = "crossgoal",
            )

        epics.size shouldBe 1

        val monitoringEpic = epics.single()

        monitoringEpic.jiraId shouldBe MONITORING_EPIC_ID
        monitoringEpic.jiraKey shouldBe MONITORING_EPIC_KEY
        monitoringEpic.jiraUrl shouldBe
            "$SIGMA_URL_PREFIX$MONITORING_EPIC_KEY"

        val tasks =
            jiraIssueRepository.findAllByAgentIdAndType(
                agentId = agent.id,
                type = JiraIssueType.task.name,
            )

        tasks.size shouldBe 2

        tasks
            .mapNotNull { it.jiraKey }
            .toSet() shouldBe
            setOf(
                QUALITY_GATE_TASK_KEY,
                DEVELOPMENT_TASK_KEY,
            )

        val qualityGateTask =
            tasks.single {
                it.jiraKey == QUALITY_GATE_TASK_KEY
            }

        qualityGateTask.parentId shouldBe monitoringEpic.id

        qualityGateTask.qualityGate?.code shouldBe
            COMPLETED_QUALITY_GATE_CODE

        qualityGateTask.project shouldBe "crossgoal"

        qualityGateTask.jiraUrl shouldBe
            "$SIGMA_URL_PREFIX$QUALITY_GATE_TASK_KEY"

        val developmentTask =
            tasks.single {
                it.jiraKey == DEVELOPMENT_TASK_KEY
            }

        developmentTask.parentId shouldBe monitoringEpic.id

        developmentTask.qualityGate?.code shouldBe
            DEVELOPMENT_STAGE_CODE

        developmentTask.project shouldBe "crossgoal"

        developmentTask.jiraUrl shouldBe
            "$SIGMA_URL_PREFIX$DEVELOPMENT_TASK_KEY"
    }

    /**
     * Проверяет SLA этапа development.
     */
    private fun assertStatusSla(
        agent: AIAgentEntity,
    ) {
        val slas =
            agentStatusSlaRepository
                .findAllByAiAgentId(agent.id)

        slas.size shouldBe 1

        val sla = slas.single()

        sla.agentStatus?.id shouldBe
            developmentStatus.id

        sla.agentStatus?.code shouldBe
            DEVELOPMENT_STATUS_CODE

        sla.plannedDate shouldBe
            LocalDateTime.of(
                2026,
                9,
                10,
                10,
                0,
            )

        /*
         * Development Task находится InProgress,
         * поэтому resolutiondate отсутствует.
         */
        sla.completedDate shouldBe null
    }

    /**
     * Вторая Jira initiative уже существовала до запуска.
     * После FR1 дубль появиться не должен.
     */
    private fun assertExistingInitiativeWasNotDuplicated() {
        val agents =
            agentRepository.findAll()

        agents.count { agent ->
            agent.agentId == EXISTING_INITIATIVE_KEY
        } shouldBe 1

        /*
         * До запуска был один existing агент,
         * после запуска появился только один новый.
         */
        agents.size shouldBe 2
    }

    /**
     * Проверяет фактически отправленные Feign-запросы.
     *
     * Так integration test проверяет не только итог в БД,
     * но и JQL, fields и пагинацию FR1.
     */
    private fun assertJiraSearchRequests() {
        val requests = jiraStub.requests()

        requests.size shouldBe 4

        val initiativeRequests =
            requests.filter { request ->
                request.jql.contains(
                    "issuetype = Инициатива"
                )
            }

        initiativeRequests.size shouldBe 2

        initiativeRequests
            .map { request -> request.startAt } shouldBe
            listOf(0, 1)

        initiativeRequests
            .forEach { request ->
                request.maxResults shouldBe PAGE_SIZE

                normalizeWhitespace(request.jql) shouldBe
                    normalizeWhitespace(
                        """
                        project = CROSSGOAL
                        AND issuetype = Инициатива
                        AND resolution = Unresolved
                        AND labels IN (AI_Native_портфель, AI-эффективность)
                        AND created >= -7d
                        ORDER BY created DESC
                        """
                    )

                request.fields shouldBe EXPECTED_INITIATIVE_FIELDS
            }

        val monitoringRequests =
            requests.filter { request ->
                request.jql.contains("\"Epic Link\"")
            }

        monitoringRequests.size shouldBe 2

        monitoringRequests
            .map { request -> request.startAt } shouldBe
            listOf(0, 1)

        monitoringRequests.forEach { request ->
            request.maxResults shouldBe PAGE_SIZE

            normalizeWhitespace(request.jql) shouldBe
                normalizeWhitespace(
                    """
                    project = CROSSGOAL
                    AND "Epic Link" = $MONITORING_EPIC_KEY
                    """
                )

            request.fields shouldBe
                EXPECTED_MONITORING_TASK_FIELDS
        }
    }

    /**
     * Читает контакты через SQL, чтобы assertion не зависел
     * от Hibernate fetch strategy entity graph.
     */
    private fun loadAgentContacts(
        agentId: Long,
    ): List<ContactRow> {

        return jdbcTemplate.query(
            """
            select
                agent_contact.type,
                contact.email,
                contact.fio,
                contact.invited
            from agent_contact
            join contact
              on contact.id = agent_contact.contact_id
            where agent_contact.agent_id = ?
            order by agent_contact.type
            """.trimIndent(),
            { resultSet, _ ->
                ContactRow(
                    type =
                        resultSet.getString("type"),
                    email =
                        resultSet.getString("email"),
                    fio =
                        resultSet.getString("fio"),
                    invited =
                        resultSet
                            .getTimestamp("invited")
                            ?.toLocalDateTime(),
                )
            },
            agentId,
        )
    }

    private fun normalizeWhitespace(
        value: String,
    ): String =
        value
            .trim()
            .replace(
                Regex("\\s+"),
                " ",
            )

    companion object {

        private val jiraStub =
            JiraSearchHttpStub()

        /*
         * Dynamic property имеет более высокий приоритет,
         * чем INTEGR_URL/application YAML.
         *
         * Поэтому настоящий Feign client приложения
         * будет обращаться в локальный test HTTP server.
         */
        @JvmStatic
        @DynamicPropertySource
        fun registerDynamicProperties(
            registry: DynamicPropertyRegistry,
        ) {
            registry.add(
                "rest.integr.base-url"
            ) {
                jiraStub.baseUrl()
            }

            registry.add(
                "scheduled.jira-sync.sigma.url.prefix"
            ) {
                SIGMA_URL_PREFIX
            }

            registry.add(
                "scheduled.jira-sync.delta.url.prefix"
            ) {
                "http://jira.test/delta/"
            }

            /*
             * Не даём scheduled wrapper самостоятельно
             * запуститься во время integration test.
             */
            registry.add(
                "scheduled.jira-sync.from-jira-new-cron"
            ) {
                "-"
            }

            /*
             * Если старые scheduler beans используют эти properties,
             * также исключаем случайный cron-запуск.
             */
            registry.add(
                "scheduled.jira-sync.cron"
            ) {
                "-"
            }

            registry.add(
                "scheduled.jira-sync.sync-update-cron"
            ) {
                "-"
            }

            /*
             * Happy-path retry не проверяет.
             *
             * Маленький delay нужен только для того,
             * чтобы ошибка самого HTTP stub не замедляла тест.
             */
            registry.add(
                "retry.maxAttempts"
            ) {
                "3"
            }

            registry.add(
                "retry.delay"
            ) {
                "1"
            }

            registry.add(
                "retry.multiplier"
            ) {
                "1.0"
            }
        }
    }
}

/**
 * Представление строки agent_contact + contact,
 * используемое только в integration assertions.
 */
private data class ContactRow(
    val type: String,
    val email: String,
    val fio: String?,
    val invited: LocalDateTime?,
)

Теперь HTTP stub. Я бы не добавлял WireMock, потому что он у вас сейчас отсутствует. Этот маленький stub использует JDK HttpServer, поэтому тестирует настоящий Feign без новой зависимости.

В тот же файл ниже теста
/**
 * Локальный HTTP stub Jira/prm-integr.
 *
 * Поднимает реальный HTTP endpoint:
 *
 * POST /internal/v1/jira/search
 *
 * Ответ выбирается на основании JQL и startAt,
 * поэтому stub одновременно позволяет проверить
 * пагинацию основного поиска и monitoring Task.
 */
private class JiraSearchHttpStub {

    private val objectMapper =
        jacksonObjectMapper()

    private val receivedRequests =
        CopyOnWriteArrayList<SearchIssueRequestDto>()

    private val executor =
        Executors.newCachedThreadPool { runnable ->
            Thread(
                runnable,
                "jira-integration-test-stub",
            ).apply {
                isDaemon = true
            }
        }

    private val server =
        HttpServer.create(
            InetSocketAddress(
                "localhost",
                0,
            ),
            0,
        )

    init {
        server.createContext(
            JIRA_SEARCH_PATH,
        ) { exchange ->
            handleSearch(exchange)
        }

        server.executor = executor
        server.start()
    }

    /**
     * Возвращает base URL, который передаётся
     * в rest.integr.base-url.
     */
    fun baseUrl(): String =
        "http://localhost:${server.address.port}"

    /**
     * Возвращает копию всех Search-запросов,
     * фактически полученных HTTP stub.
     */
    fun requests(): List<SearchIssueRequestDto> =
        receivedRequests.toList()

    /**
     * Очищает историю запросов между тестами.
     */
    fun reset() {
        receivedRequests.clear()
    }

    /**
     * Останавливает HTTP stub после завершения spec.
     */
    fun stop() {
        server.stop(0)
        executor.shutdownNow()
    }

    private fun handleSearch(
        exchange: HttpExchange,
    ) {
        try {
            if (
                exchange.requestMethod != "POST"
            ) {
                sendTextResponse(
                    exchange = exchange,
                    status = 405,
                    body = "Only POST is supported",
                )

                return
            }

            val requestBody =
                exchange.requestBody
                    .bufferedReader(
                        StandardCharsets.UTF_8
                    )
                    .use { reader ->
                        reader.readText()
                    }

            val request =
                objectMapper.readValue(
                    requestBody,
                    SearchIssueRequestDto::class.java,
                )

            receivedRequests += request

            val response =
                when {
                    isNewInitiativesRequest(request) ->
                        newInitiativesResponse(
                            request = request
                        )

                    isMonitoringTasksRequest(request) ->
                        monitoringTasksResponse(
                            request = request
                        )

                    else ->
                        error(
                            "Unexpected Jira Search request: " +
                                "jql='${request.jql}', " +
                                "startAt=${request.startAt}"
                        )
                }

            sendJsonResponse(
                exchange = exchange,
                response = response,
            )

        } catch (exception: Exception) {
            sendTextResponse(
                exchange = exchange,
                status = 500,
                body =
                    "Jira test stub failed: " +
                        exception.message,
            )
        }
    }

    private fun isNewInitiativesRequest(
        request: SearchIssueRequestDto,
    ): Boolean =
        request.jql.contains(
            "issuetype = Инициатива"
        )

    private fun isMonitoringTasksRequest(
        request: SearchIssueRequestDto,
    ): Boolean =
        request.jql.contains("\"Epic Link\"") &&
            request.jql.contains(MONITORING_EPIC_KEY)

    /**
     * Эмулирует две страницы поиска инициатив:
     *
     * startAt=0 -> новая CROSSGOAL-100
     * startAt=1 -> уже существующая CROSSGOAL-200
     */
    private fun newInitiativesResponse(
        request: SearchIssueRequestDto,
    ): SearchIssueResponseDto {

        return when (
            request.startAt ?: 0
        ) {
            0 ->
                SearchIssueResponseDto(
                    startAt = 0,
                    maxResults = PAGE_SIZE,
                    total = 2,
                    issues =
                        listOf(
                            newInitiative()
                        ),
                )

            1 ->
                SearchIssueResponseDto(
                    startAt = 1,
                    maxResults = PAGE_SIZE,
                    total = 2,
                    issues =
                        listOf(
                            existingInitiative()
                        ),
                )

            else ->
                SearchIssueResponseDto(
                    startAt = request.startAt,
                    maxResults = PAGE_SIZE,
                    total = 2,
                    issues = emptyList(),
                )
        }
    }

    /**
     * Эмулирует две страницы Task monitoring epic:
     *
     * startAt=0 -> завершённая quality_gate Task
     * startAt=1 -> development Task в InProgress
     */
    private fun monitoringTasksResponse(
        request: SearchIssueRequestDto,
    ): SearchIssueResponseDto {

        return when (
            request.startAt ?: 0
        ) {
            0 ->
                SearchIssueResponseDto(
                    startAt = 0,
                    maxResults = PAGE_SIZE,
                    total = 2,
                    issues =
                        listOf(
                            completedQualityGateTask()
                        ),
                )

            1 ->
                SearchIssueResponseDto(
                    startAt = 1,
                    maxResults = PAGE_SIZE,
                    total = 2,
                    issues =
                        listOf(
                            developmentTask()
                        ),
                )

            else ->
                SearchIssueResponseDto(
                    startAt = request.startAt,
                    maxResults = PAGE_SIZE,
                    total = 2,
                    issues = emptyList(),
                )
        }
    }

    /**
     * Новая Jira initiative.
     *
     * Payload намеренно содержит данные,
     * необходимые всем этапам FR1.
     */
    private fun newInitiative(): SearchIssueDto {

        return SearchIssueDto(
            id = NEW_INITIATIVE_JIRA_ID,
            key = NEW_INITIATIVE_KEY,
            fields =
                SearchIssueFieldsDto(
                    summary = NEW_INITIATIVE_SUMMARY,
                    description = LONG_DESCRIPTION,

                    status =
                        SearchIssueStatusDto(
                            id = "3",
                            name = "В работе",
                        ),

                    labels =
                        listOf(
                            "AI_Native_портфель",
                            "AI-агент",
                        ),

                    /*
                     * Значение не равно label буквально,
                     * а СОДЕРЖИТ division label.
                     *
                     * Тем самым integration test также проверяет
                     * актуальное правило organization resolver.
                     */
                    customfield_30001 =
                        listOf(
                            "Исполнитель / " +
                                TEST_DIVISION_LABEL +
                                " / команда"
                        ),

                    customfield_34300 =
                        "Оптимизация: 123,45",

                    customfield_30401 =
                        "Доход 678.90 руб.",

                    /*
                     * Один involved resource.
                     */
                    customfield_31304 =
                        "12,5 FTE",

                    assignee =
                        SearchIssueUserInfoDto(
                            emailAddress =
                                DEVELOPER_EMAIL,
                            displayName =
                                DEVELOPER_NAME,
                            active = true,
                        ),

                    reporter =
                        SearchIssueUserInfoDto(
                            emailAddress =
                                CUSTOMER_EMAIL,
                            displayName =
                                CUSTOMER_NAME,
                            active = true,
                        ),

                    /*
                     * Поле заполнено специально.
                     *
                     * FR1 должен использовать reporter,
                     * потому что он имеет приоритет.
                     */
                    customfield_29202 =
                        SearchIssueUserInfoDto(
                            emailAddress =
                                FALLBACK_CUSTOMER_EMAIL,
                            displayName =
                                "Fallback Customer",
                            active = true,
                        ),

                    /*
                     * Имя отличается регистром и пробелами
                     * от значения справочника.
                     */
                    customfield_15903 =
                        listOf(
                            SearchIssueCheckboxOptionDto(
                                id = 1,
                                name = "  GIGA    Chat ",
                                checked = true,
                            )
                        ),

                    issuelinks =
                        listOf(
                            strategyLink(),
                            firstGigaUsageLink(),
                            secondGigaUsageLink(),
                            monitoringEpicLink(),
                        ),

                    created =
                        "2026-08-25T10:00:00.000+0300",

                    updated =
                        "2026-08-25T11:00:00.000+0300",
                ),
        )
    }

    /**
     * Инициатива второй страницы.
     *
     * Она уже присутствует в БД и должна
     * быть пропущена до creation/monitoring.
     */
    private fun existingInitiative(): SearchIssueDto =
        SearchIssueDto(
            id = EXISTING_INITIATIVE_JIRA_ID,
            key = EXISTING_INITIATIVE_KEY,
            fields =
                SearchIssueFieldsDto(
                    summary =
                        "Already existing initiative",
                    status =
                        SearchIssueStatusDto(
                            id = "3",
                            name = "В работе",
                        ),
                    labels =
                        listOf(
                            "AI_Native_портфель"
                        ),
                ),
        )

    private fun strategyLink(): SearchIssueLinkDto =
        SearchIssueLinkDto(
            id = "link-strategy",
            outwardIssue =
                SearchLinkedIssueDto(
                    id = "strategy-id",
                    key = STRATEGY_JIRA_KEY,
                    fields =
                        SearchLinkedIssueFieldsDto(
                            summary =
                                "Стратегия 2026"
                        ),
                ),
        )

    private fun firstGigaUsageLink():
        SearchIssueLinkDto =
        SearchIssueLinkDto(
            id = "link-giga-1",
            outwardIssue =
                SearchLinkedIssueDto(
                    id = FIRST_GIGAUSAGE_ID,
                    key = FIRST_GIGAUSAGE_KEY,
                ),
        )

    private fun secondGigaUsageLink():
        SearchIssueLinkDto =
        SearchIssueLinkDto(
            id = "link-giga-2",
            inwardIssue =
                SearchLinkedIssueDto(
                    id = SECOND_GIGAUSAGE_ID,
                    key = SECOND_GIGAUSAGE_KEY,
                ),
        )

    private fun monitoringEpicLink():
        SearchIssueLinkDto =
        SearchIssueLinkDto(
            id = "link-monitoring",
            outwardIssue =
                SearchLinkedIssueDto(
                    id = MONITORING_EPIC_ID,
                    key = MONITORING_EPIC_KEY,
                    fields =
                        SearchLinkedIssueFieldsDto(
                            summary =
                                "Мониторинг портфеля AI-Native"
                        ),
                ),
        )

    /**
     * Завершённая Task вехи.
     *
     * status.id=10110 относится к checked.
     */
    private fun completedQualityGateTask():
        SearchIssueDto =
        SearchIssueDto(
            id = QUALITY_GATE_TASK_ID,
            key = QUALITY_GATE_TASK_KEY,
            fields =
                SearchIssueFieldsDto(
                    summary =
                        "Архитектура: согласование",
                    status =
                        SearchIssueStatusDto(
                            id = "10110",
                            name = "Done",
                        ),
                    resolutiondate =
                        "2026-08-20T12:00:00.000+0300",
                ),
        )

    /**
     * Task этапа development.
     *
     * status.id=3 => InProgress.
     *
     * Именно она должна определить итоговый
     * agent_status=development и создать SLA.
     */
    private fun developmentTask():
        SearchIssueDto =
        SearchIssueDto(
            id = DEVELOPMENT_TASK_ID,
            key = DEVELOPMENT_TASK_KEY,
            fields =
                SearchIssueFieldsDto(
                    summary =
                        "Этап: разработка",
                    status =
                        SearchIssueStatusDto(
                            id = "3",
                            name = "In Progress",
                        ),
                    customfield_16701 =
                        PLANNED_DEVELOPMENT_DATE,
                    resolutiondate = null,
                ),
        )

    private fun sendJsonResponse(
        exchange: HttpExchange,
        response: SearchIssueResponseDto,
    ) {
        val bytes =
            objectMapper.writeValueAsBytes(response)

        exchange.responseHeaders.add(
            "Content-Type",
            "application/json;charset=UTF-8",
        )

        exchange.sendResponseHeaders(
            200,
            bytes.size.toLong(),
        )

        exchange.responseBody.use { body ->
            body.write(bytes)
        }
    }

    private fun sendTextResponse(
        exchange: HttpExchange,
        status: Int,
        body: String,
    ) {
        val bytes =
            body.toByteArray(
                StandardCharsets.UTF_8
            )

        exchange.responseHeaders.add(
            "Content-Type",
            "text/plain;charset=UTF-8",
        )

        exchange.sendResponseHeaders(
            status,
            bytes.size.toLong(),
        )

        exchange.responseBody.use { responseBody ->
            responseBody.write(bytes)
        }
    }

    companion object {
        private const val JIRA_SEARCH_PATH =
            "/internal/v1/jira/search"
    }
}

private const val NEW_DEPTH = 7
private const val PAGE_SIZE = 1

private const val NEW_INITIATIVE_KEY =
    "CROSSGOAL-100"

private const val NEW_INITIATIVE_JIRA_ID =
    "10001"

private const val EXISTING_INITIATIVE_KEY =
    "CROSSGOAL-200"

private const val EXISTING_INITIATIVE_JIRA_ID =
    "10002"

private const val NEW_INITIATIVE_SUMMARY =
    "Integration FR1 AI-agent"

private const val MAX_DESCRIPTION_LENGTH =
    2000

private val LONG_DESCRIPTION =
    "D".repeat(2100)

private const val ANALYSIS_STATUS_CODE =
    "analysis"

private const val DEVELOPMENT_STATUS_CODE =
    "development"

private const val TARGET_SOLUTION_STATUS_CODE =
    "targetSolution"

private const val TEST_BLOCK_CODE =
    "test-block"

private const val TEST_BLOCK_LABEL =
    "Блок Integration"

private const val TEST_DIVISION_CODE =
    "test-division"

private const val TEST_DIVISION_LABEL =
    "Трайб Integration"

private const val AGENT_INITIATIVE_TYPE_CODE =
    "agent"

private const val GENERATIVE_AI_INITIATIVE_TYPE_CODE =
    "genAiSolution"

private const val STRATEGY_JIRA_KEY =
    "STRATEGY-2026"

private const val ENABLER_DICTIONARY_NAME =
    "Giga Chat"

private const val COMPLETED_QUALITY_GATE_CODE =
    "QG_ARCHITECTURE"

private const val UNCHECKED_QUALITY_GATE_CODE =
    "QG_SECURITY"

private const val DEVELOPMENT_STAGE_CODE =
    "STAGE_DEVELOPMENT"

private const val DEVELOPER_EMAIL =
    "developer@sberbank.ru"

private const val DEVELOPER_NAME =
    "Developer Integration"

private const val CUSTOMER_EMAIL =
    "customer@sber.ru"

private const val CUSTOMER_NAME =
    "Customer Integration"

private const val FALLBACK_CUSTOMER_EMAIL =
    "fallback@sber.ru"

private const val FIRST_GIGAUSAGE_KEY =
    "GIGAUSAGE-100"

private const val FIRST_GIGAUSAGE_ID =
    "9100"

private const val SECOND_GIGAUSAGE_KEY =
    "GIGAUSAGE-200"

private const val SECOND_GIGAUSAGE_ID =
    "9200"

private const val MONITORING_EPIC_KEY =
    "CROSSGOAL-500"

private const val MONITORING_EPIC_ID =
    "50000"

private const val QUALITY_GATE_TASK_KEY =
    "CROSSGOAL-501"

private const val QUALITY_GATE_TASK_ID =
    "50100"

private const val DEVELOPMENT_TASK_KEY =
    "CROSSGOAL-502"

private const val DEVELOPMENT_TASK_ID =
    "50200"

private const val PLANNED_DEVELOPMENT_DATE =
    "2026-09-10T10:00:00.000+0300"

private const val SIGMA_URL_PREFIX =
    "http://jira.test/browse/"

/**
 * Поля основного Jira Search.
 *
 * Список намеренно повторяет контракт FR1:
 * integration test должен заметить случайное удаление поля.
 */
private val EXPECTED_INITIATIVE_FIELDS =
    listOf(
        "summary",
        "description",
        "status",
        "labels",
        "customfield_30000",
        "customfield_30001",
        "customfield_30002",
        "customfield_34300",
        "customfield_30401",
        "customfield_31304",
        "customfield_31305",
        "customfield_31306",
        "customfield_31307",
        "issuelinks",
        "customfield_15903",
        "assignee",
        "reporter",
        "customfield_29202",
        "customfield_29203",
        "customfield_29205",
        "lastViewed",
        "resolutiondate",
        "created",
        "updated",
    )

private val EXPECTED_MONITORING_TASK_FIELDS =
    listOf(
        "summary",
        "description",
        "status",
        "customfield_16700",
        "customfield_16701",
        "assignee",
        "reporter",
        "lastViewed",
        "resolutiondate",
        "created",
        "updated",
    )

```
