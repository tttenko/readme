```java
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.sun.net.httpserver.HttpExchange
import com.sun.net.httpserver.HttpServer
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.test.annotation.DirtiesContext
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import java.math.BigDecimal
import java.net.InetSocketAddress
import java.nio.charset.StandardCharsets
import java.sql.ResultSet
import java.time.LocalDateTime
import java.util.concurrent.CopyOnWriteArrayList
import java.util.concurrent.Executors

// ============================================================================
// PROJECT IMPORTS
// ============================================================================
//
// Добавь фактические package через Alt+Enter:
//
// AIAgentEntity
// BlockEntity
// DivisionEntity
// EnablerEntity
// InitiativeTypeEntity
// OptionsEntity
// QualityGateEntity
// QualityGateType
// StatusEntity
// StrategyEntity
//
// AIAgentRepository
// BlockRepository
// DivisionRepository
// EnablerRepository
// InitiativeTypeRepository
// OptionsRepository
// QualityGateRepository
// StatusRepository
// StrategyRepository
//
// JiraNewInitiativeImportService
//
// SearchIssueRequestDto
// SearchIssueResponseDto
// SearchIssueDto
// SearchIssueFieldsDto
// SearchIssueStatusDto
// SearchIssueLinkDto
// SearchLinkedIssueDto
// SearchLinkedIssueFieldsDto
// SearchIssueUserInfoDto
// SearchIssueCheckboxOptionDto
//
// AbstractJUnitIntegrationTest
// ============================================================================

/**
 * Сквозной integration test FR1.
 *
 * Проверяет совместную работу всех пяти этапов:
 *
 * - основной Jira Search;
 * - пагинацию;
 * - проверку существующей инициативы;
 * - создание новой инициативы;
 * - стратегии;
 * - involved resources;
 * - контакты;
 * - enablers;
 * - initial quality gates;
 * - CROSSGOAL/GIGAUSAGE jira_issue;
 * - monitoring epic;
 * - Jira Search Task;
 * - Task -> quality gate matching;
 * - agent_quality_gate;
 * - SLA;
 * - вычисление итогового статуса;
 * - jira_issue Task;
 * - jira_from_status=done.
 *
 * Все внутренние Spring-компоненты FR1 реальные.
 * Подменяется только внешний prm-integr/Jira HTTP endpoint.
 */
@ActiveProfiles("integration-test")
@DirtiesContext(
    classMode = DirtiesContext.ClassMode.AFTER_CLASS,
)
class JiraNewInitiativeImportIntegrationTest :
    AbstractJUnitIntegrationTest() {

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
    private lateinit var jdbcTemplate:
        JdbcTemplate

    private lateinit var analysisStatus: StatusEntity
    private lateinit var developmentStatus: StatusEntity

    private lateinit var testBlock: BlockEntity
    private lateinit var testDivision: DivisionEntity

    private lateinit var strategy: StrategyEntity
    private lateinit var enabler: EnablerEntity

    @BeforeEach
    fun setUp() {
        /*
         * Сам тест НЕ @Transactional.
         *
         * Production-код использует REQUIRES_NEW,
         * поэтому fixture должен быть закоммичен.
         */
        clearTables()

        jiraStub.reset()

        prepareReferenceData()
        prepareExistingInitiative()
    }

    @AfterEach
    fun tearDown() {
        clearTables()
        jiraStub.reset()
    }

    /**
     * Главный smoke/integration test FR1.
     */
    @Test
    fun `should import new Jira initiative with all related monitoring data`() {
        // WHEN
        jiraNewInitiativeImportService.importNewInitiatives()

        // THEN
        val agent =
            agentRepository.findFirstByAgentId(
                NEW_INITIATIVE_KEY
            )

        assertThat(agent)
            .describedAs(
                "New Jira initiative must be created"
            )
            .isNotNull

        val createdAgent = requireNotNull(agent)

        assertAgent(createdAgent)
        assertStrategy(createdAgent)
        assertInvolvedResource(createdAgent)
        assertContacts(createdAgent)
        assertEnablers(createdAgent)
        assertQualityGates(createdAgent)
        assertJiraIssues(createdAgent)
        assertStatusSla(createdAgent)

        assertExistingInitiativeWasNotDuplicated()

        assertJiraSearchRequests()
    }

    // =========================================================================
    // FIXTURE
    // =========================================================================

    /**
     * Создаёт минимальный набор справочников,
     * необходимых полному happy-path FR1.
     *
     * maxResults=1 выставлен специально,
     * чтобы проверить пагинацию и для инициатив,
     * и для monitoring Task.
     */
    private fun prepareReferenceData() {
        prepareOptions()
        prepareStatuses()
        prepareOrganization()
        prepareInitiativeTypes()
        prepareStrategy()
        prepareEnabler()
        prepareQualityGates()
    }

    private fun prepareOptions() {
        optionsRepository.saveAndFlush(
            OptionsEntity().apply {
                newDepth = NEW_DEPTH
                updateDepth = 3
                maxResults = PAGE_SIZE
            }
        )
    }

    private fun prepareStatuses() {
        analysisStatus =
            statusRepository.saveAndFlush(
                StatusEntity().apply {
                    name = "Концепция"
                    code = ANALYSIS_STATUS_CODE
                    ordering = 10L
                    disabled = false
                }
            )

        developmentStatus =
            statusRepository.saveAndFlush(
                StatusEntity().apply {
                    name = "Разработка"
                    code = DEVELOPMENT_STATUS_CODE
                    ordering = 20L
                    disabled = false
                }
            )

        statusRepository.saveAndFlush(
            StatusEntity().apply {
                name = "Целевое решение"
                code = TARGET_SOLUTION_STATUS_CODE
                ordering = 30L
                disabled = false
            }
        )
    }

    private fun prepareOrganization() {
        testBlock =
            blockRepository.saveAndFlush(
                BlockEntity().apply {
                    code = TEST_BLOCK_CODE
                    shortName = TEST_BLOCK_SHORT_NAME
                    name = TEST_BLOCK_NAME
                    label = TEST_BLOCK_LABEL
                    disabled = false
                }
            )

        testDivision =
            divisionRepository.saveAndFlush(
                DivisionEntity().apply {
                    block = testBlock

                    code = TEST_DIVISION_CODE
                    shortName = TEST_DIVISION_SHORT_NAME
                    name = TEST_DIVISION_NAME

                    label = TEST_DIVISION_LABEL
                    ordering = 10L
                    disabled = false
                }
            )
    }

    private fun prepareInitiativeTypes() {
        initiativeTypeRepository.saveAndFlush(
            InitiativeTypeEntity(
                code = AGENT_INITIATIVE_TYPE,
                name = "AI-агент",
                description = "AI agent",
            )
        )

        initiativeTypeRepository.saveAndFlush(
            InitiativeTypeEntity(
                code = GEN_AI_INITIATIVE_TYPE,
                name = "GenAI solution",
                description = "Generative AI solution",
            )
        )
    }

    private fun prepareStrategy() {
        strategy =
            strategyRepository.saveAndFlush(
                StrategyEntity(
                    jiraIssue = STRATEGY_JIRA_KEY,
                    name = "Стратегия Integration Test",
                )
            )
    }

    private fun prepareEnabler() {
        enabler =
            enablerRepository.saveAndFlush(
                EnablerEntity().apply {
                    name = ENABLER_DICTIONARY_NAME
                    shortDescription =
                        "Integration test enabler"

                    description =
                        "Enabler used by FR1 integration test"

                    disabled = false
                }
            )
    }

    private fun prepareQualityGates() {
        /*
         * Веха.
         *
         * Jira Task будет в status=Done,
         * поэтому после monitoring:
         *
         * agent_quality_gate.state = checked
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
         * Активная веха без соответствующей Jira Task.
         *
         * После создания инициативы должна остаться unchecked.
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
         * Этап development.
         *
         * Его Jira Task будет иметь status.id=3 (InProgress),
         * поэтому итоговый статус инициативы станет development.
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
     * Эта инициатива будет возвращена Jira
     * на второй странице основного Search.
     *
     * Scheduler обязан определить, что она уже есть,
     * и не создать ещё одну запись.
     */
    private fun prepareExistingInitiative() {
        agentRepository.saveAndFlush(
            AIAgentEntity(
                agentId = EXISTING_INITIATIVE_KEY,
                agentName = "Existing initiative",
                agentJiraUrl = EXISTING_INITIATIVE_KEY,
            ).apply {
                disabled = false
                created = LocalDateTime.now()
            }
        )
    }

    // =========================================================================
    // ASSERT: AI_AGENT
    // =========================================================================

    private fun assertAgent(
        agent: AIAgentEntity,
    ) {
        assertThat(agent.agentId)
            .isEqualTo(NEW_INITIATIVE_KEY)

        assertThat(agent.agentName)
            .isEqualTo(NEW_INITIATIVE_SUMMARY)

        /*
         * Актуальное ограничение проекта = 2000.
         */
        assertThat(agent.agentDescription)
            .hasSize(MAX_DESCRIPTION_LENGTH)

        assertThat(agent.agentDescription)
            .isEqualTo(
                LONG_DESCRIPTION.take(
                    MAX_DESCRIPTION_LENGTH
                )
            )

        assertThat(agent.agentJiraUrl)
            .isEqualTo(NEW_INITIATIVE_KEY)

        assertThat(agent.agentEffectOptimization)
            .isEqualByComparingTo(
                BigDecimal("123.45")
            )

        assertThat(agent.agentEffectRevenue)
            .isEqualByComparingTo(
                BigDecimal("678.90")
            )

        assertThat(agent.block?.id)
            .isEqualTo(testBlock.id)

        assertThat(agent.division?.id)
            .isEqualTo(testDivision.id)

        assertThat(agent.initiativeType?.code)
            .isEqualTo(AGENT_INITIATIVE_TYPE)

        /*
         * Изначально ставится analysis.
         *
         * Monitoring Task этапа development
         * находится в Jira status=3 (InProgress),
         * поэтому после этапа 4 статус должен измениться.
         */
        assertThat(agent.agentStatus?.code)
            .isEqualTo(DEVELOPMENT_STATUS_CODE)

        assertThat(agent.importStatus)
            .isEqualTo("blocked")

        assertThat(agent.jiraFromStatus)
            .isEqualTo("done")

        assertThat(agent.disabled)
            .isFalse()

        assertThat(agent.created)
            .isNotNull

        assertThat(agent.updated)
            .isNotNull

        assertThat(agent.jiraUpdated)
            .isNotNull
    }

    // =========================================================================
    // ASSERT: STRATEGY
    // =========================================================================

    private fun assertStrategy(
        agent: AIAgentEntity,
    ) {
        val rows =
            jdbcTemplate.query(
                """
                    select
                        strategy_id,
                        jira_link
                    from agent_strategy
                    where ai_agent_id = ?
                """.trimIndent(),
                { resultSet, _ ->
                    StrategyRelationRow(
                        strategyId =
                            resultSet.getLong(
                                "strategy_id"
                            ),
                        jiraLink =
                            resultSet.getString(
                                "jira_link"
                            ),
                    )
                },
                agent.id,
            )

        assertThat(rows)
            .hasSize(1)

        val row = rows.single()

        assertThat(row.strategyId)
            .isEqualTo(strategy.id)

        assertThat(row.jiraLink)
            .isEqualTo("done")
    }

    // =========================================================================
    // ASSERT: INVOLVED RESOURCE
    // =========================================================================

    private fun assertInvolvedResource(
        agent: AIAgentEntity,
    ) {
        val rows =
            jdbcTemplate.query(
                """
                    select
                        source,
                        type,
                        value,
                        time_allocated,
                        created,
                        updated
                    from involved_resource
                    where ai_agent_id = ?
                """.trimIndent(),
                { resultSet, _ ->
                    InvolvedResourceRow(
                        source =
                            resultSet.getString("source"),
                        type =
                            resultSet.getString("type"),
                        value =
                            resultSet.getBigDecimal("value"),
                        timeAllocated =
                            resultSet.getString(
                                "time_allocated"
                            ),
                        created =
                            resultSet
                                .getTimestamp("created")
                                ?.toLocalDateTime(),
                        updated =
                            resultSet
                                .getTimestamp("updated")
                                ?.toLocalDateTime(),
                    )
                },
                agent.id,
            )

        assertThat(rows)
            .hasSize(1)

        val resource = rows.single()

        assertThat(resource.source)
            .isEqualTo("without_steerCo")

        assertThat(resource.type)
            .isEqualTo("business")

        assertThat(resource.value)
            .isEqualByComparingTo(
                BigDecimal("12.5")
            )

        assertThat(resource.timeAllocated)
            .isNull()

        assertThat(resource.created)
            .isNotNull

        assertThat(resource.updated)
            .isNotNull
    }

    // =========================================================================
    // ASSERT: CONTACTS
    // =========================================================================

    private fun assertContacts(
        agent: AIAgentEntity,
    ) {
        val contacts =
            jdbcTemplate.query(
                """
                    select
                        ac.type,
                        c.email,
                        c.fio,
                        c.invited
                    from agent_contact ac
                    join contact c
                      on c.id = ac.contact_id
                    where ac.agent_id = ?
                    order by ac.type
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
                agent.id,
            )

        assertThat(contacts)
            .hasSize(2)

        assertThat(
            contacts.map {
                it.type to it.email
            }
        )
            .containsExactlyInAnyOrder(
                "developer" to DEVELOPER_EMAIL,
                "customer" to CUSTOMER_EMAIL,
            )

        val developer =
            contacts.single {
                it.type == "developer"
            }

        assertThat(developer.email)
            .isEqualTo(DEVELOPER_EMAIL)

        assertThat(developer.fio)
            .isEqualTo(DEVELOPER_NAME)

        /*
         * Jira-import не должен отправлять приглашение.
         */
        assertThat(developer.invited)
            .isNull()

        val customer =
            contacts.single {
                it.type == "customer"
            }

        assertThat(customer.email)
            .isEqualTo(CUSTOMER_EMAIL)

        assertThat(customer.fio)
            .isEqualTo(CUSTOMER_NAME)

        assertThat(customer.invited)
            .isNull()

        /*
         * Reporter заполнен, поэтому customfield_29202
         * использоваться не должен.
         */
        val fallbackCount =
            jdbcTemplate.queryForObject(
                """
                    select count(*)
                    from contact
                    where email = ?
                """.trimIndent(),
                Long::class.java,
                FALLBACK_CUSTOMER_EMAIL,
            )

        assertThat(fallbackCount)
            .isZero()
    }

    // =========================================================================
    // ASSERT: ENABLER
    // =========================================================================

    private fun assertEnablers(
        agent: AIAgentEntity,
    ) {
        val enablerIds =
            jdbcTemplate.queryForList(
                """
                    select enabler_id
                    from agent_enabler
                    where agent_id = ?
                """.trimIndent(),
                Long::class.java,
                agent.id,
            )

        assertThat(enablerIds)
            .containsExactly(enabler.id)
    }

    // =========================================================================
    // ASSERT: QUALITY GATES
    // =========================================================================

    private fun assertQualityGates(
        agent: AIAgentEntity,
    ) {
        val states =
            jdbcTemplate.query(
                """
                    select
                        quality_gate_code,
                        state
                    from agent_quality_gate
                    where ai_agent_id = ?
                """.trimIndent(),
                { resultSet, _ ->
                    resultSet.getString(
                        "quality_gate_code"
                    ) to resultSet.getString(
                        "state"
                    )
                },
                agent.id,
            ).toMap()

        /*
         * Только type=quality_gate создаётся
         * в agent_quality_gate.
         *
         * type=status сюда не входит.
         */
        assertThat(states)
            .hasSize(2)

        assertThat(
            states[COMPLETED_QUALITY_GATE_CODE]
        )
            .isEqualTo("checked")

        assertThat(
            states[UNCHECKED_QUALITY_GATE_CODE]
        )
            .isEqualTo("unchecked")
    }

    // =========================================================================
    // ASSERT: JIRA ISSUES
    // =========================================================================

    private fun assertJiraIssues(
        agent: AIAgentEntity,
    ) {
        val issues =
            jdbcTemplate.query(
                """
                    select
                        id,
                        type,
                        project,
                        parent_id,
                        jira_id,
                        jira_key,
                        jira_url,
                        quality_gate_code
                    from jira_issue
                    where agent_id = ?
                    order by id
                """.trimIndent(),
                { resultSet, _ ->
                    jiraIssueRow(resultSet)
                },
                agent.id,
            )

        /*
         * Ожидаем:
         *
         * 1 CROSSGOAL initiative
         * 2 GIGAUSAGE initiative
         * 1 monitoring epic
         * 2 monitoring task
         */
        assertThat(issues)
            .hasSize(6)

        assertCrossgoalInitiative(issues)
        assertGigaUsageIssues(issues)
        assertMonitoringEpicAndTasks(issues)
    }

    private fun assertCrossgoalInitiative(
        issues: List<JiraIssueRow>,
    ) {
        val initiative =
            issues.single {
                it.project == "crossgoal" &&
                    it.type == "initiative"
            }

        assertThat(initiative.jiraId)
            .isEqualTo(NEW_INITIATIVE_JIRA_ID)

        assertThat(initiative.jiraKey)
            .isEqualTo(NEW_INITIATIVE_KEY)

        assertThat(initiative.jiraUrl)
            .isEqualTo(
                "$SIGMA_URL_PREFIX$NEW_INITIATIVE_KEY"
            )

        assertThat(initiative.parentId)
            .isNull()

        assertThat(initiative.qualityGateCode)
            .isNull()
    }

    private fun assertGigaUsageIssues(
        issues: List<JiraIssueRow>,
    ) {
        val gigaUsageIssues =
            issues.filter {
                it.project == "gigausage" &&
                    it.type == "initiative"
            }

        assertThat(gigaUsageIssues)
            .hasSize(2)

        assertThat(
            gigaUsageIssues.map {
                it.jiraKey
            }
        )
            .containsExactlyInAnyOrder(
                FIRST_GIGAUSAGE_KEY,
                SECOND_GIGAUSAGE_KEY,
            )

        val first =
            gigaUsageIssues.single {
                it.jiraKey ==
                    FIRST_GIGAUSAGE_KEY
            }

        assertThat(first.jiraId)
            .isEqualTo(FIRST_GIGAUSAGE_ID)

        assertThat(first.jiraUrl)
            .isEqualTo(
                "$SIGMA_URL_PREFIX$FIRST_GIGAUSAGE_KEY"
            )

        val second =
            gigaUsageIssues.single {
                it.jiraKey ==
                    SECOND_GIGAUSAGE_KEY
            }

        assertThat(second.jiraId)
            .isEqualTo(SECOND_GIGAUSAGE_ID)

        assertThat(second.jiraUrl)
            .isEqualTo(
                "$SIGMA_URL_PREFIX$SECOND_GIGAUSAGE_KEY"
            )
    }

    private fun assertMonitoringEpicAndTasks(
        issues: List<JiraIssueRow>,
    ) {
        val epic =
            issues.single {
                it.project == "crossgoal" &&
                    it.type == "epic"
            }

        assertThat(epic.jiraId)
            .isEqualTo(MONITORING_EPIC_ID)

        assertThat(epic.jiraKey)
            .isEqualTo(MONITORING_EPIC_KEY)

        assertThat(epic.jiraUrl)
            .isEqualTo(
                "$SIGMA_URL_PREFIX$MONITORING_EPIC_KEY"
            )

        val tasks =
            issues.filter {
                it.project == "crossgoal" &&
                    it.type == "task"
            }

        assertThat(tasks)
            .hasSize(2)

        assertThat(
            tasks.map {
                it.jiraKey
            }
        )
            .containsExactlyInAnyOrder(
                QUALITY_GATE_TASK_KEY,
                DEVELOPMENT_TASK_KEY,
            )

        val qualityGateTask =
            tasks.single {
                it.jiraKey ==
                    QUALITY_GATE_TASK_KEY
            }

        assertThat(qualityGateTask.parentId)
            .isEqualTo(epic.id)

        assertThat(
            qualityGateTask.qualityGateCode
        )
            .isEqualTo(
                COMPLETED_QUALITY_GATE_CODE
            )

        assertThat(qualityGateTask.jiraId)
            .isEqualTo(QUALITY_GATE_TASK_ID)

        assertThat(qualityGateTask.jiraUrl)
            .isEqualTo(
                "$SIGMA_URL_PREFIX$QUALITY_GATE_TASK_KEY"
            )

        val developmentTask =
            tasks.single {
                it.jiraKey ==
                    DEVELOPMENT_TASK_KEY
            }

        assertThat(developmentTask.parentId)
            .isEqualTo(epic.id)

        assertThat(
            developmentTask.qualityGateCode
        )
            .isEqualTo(
                DEVELOPMENT_STAGE_CODE
            )

        assertThat(developmentTask.jiraId)
            .isEqualTo(DEVELOPMENT_TASK_ID)

        assertThat(developmentTask.jiraUrl)
            .isEqualTo(
                "$SIGMA_URL_PREFIX$DEVELOPMENT_TASK_KEY"
            )
    }

    // =========================================================================
    // ASSERT: SLA
    // =========================================================================

    private fun assertStatusSla(
        agent: AIAgentEntity,
    ) {
        val rows =
            jdbcTemplate.query(
                """
                    select
                        s.code,
                        ass.planned_date,
                        ass.completed_date
                    from agent_status_sla ass
                    join status s
                      on s.id = ass.agent_status_id
                    where ass.ai_agent_id = ?
                """.trimIndent(),
                { resultSet, _ ->
                    StatusSlaRow(
                        statusCode =
                            resultSet.getString("code"),
                        plannedDate =
                            resultSet
                                .getTimestamp(
                                    "planned_date"
                                )
                                ?.toLocalDateTime(),
                        completedDate =
                            resultSet
                                .getTimestamp(
                                    "completed_date"
                                )
                                ?.toLocalDateTime(),
                    )
                },
                agent.id,
            )

        assertThat(rows)
            .hasSize(1)

        val developmentSla =
            rows.single()

        assertThat(
            developmentSla.statusCode
        )
            .isEqualTo(
                DEVELOPMENT_STATUS_CODE
            )

        assertThat(
            developmentSla.plannedDate
        )
            .isEqualTo(
                LocalDateTime.of(
                    2026,
                    9,
                    10,
                    10,
                    0,
                )
            )

        /*
         * Task сейчас InProgress,
         * resolutiondate отсутствует.
         */
        assertThat(
            developmentSla.completedDate
        )
            .isNull()
    }

    // =========================================================================
    // ASSERT: EXISTING INITIATIVE
    // =========================================================================

    private fun assertExistingInitiativeWasNotDuplicated() {
        val existingCount =
            jdbcTemplate.queryForObject(
                """
                    select count(*)
                    from ai_agent
                    where agent_id = ?
                """.trimIndent(),
                Long::class.java,
                EXISTING_INITIATIVE_KEY,
            )

        assertThat(existingCount)
            .isEqualTo(1L)

        val newCount =
            jdbcTemplate.queryForObject(
                """
                    select count(*)
                    from ai_agent
                    where agent_id = ?
                """.trimIndent(),
                Long::class.java,
                NEW_INITIATIVE_KEY,
            )

        assertThat(newCount)
            .isEqualTo(1L)

        /*
         * Перед запуском был один агент.
         * FR1 должен добавить ровно одного нового.
         */
        val total =
            jdbcTemplate.queryForObject(
                """
                    select count(*)
                    from ai_agent
                """.trimIndent(),
                Long::class.java,
            )

        assertThat(total)
            .isEqualTo(2L)
    }

    // =========================================================================
    // ASSERT: HTTP REQUESTS
    // =========================================================================

    /**
     * Проверяем реальные HTTP-запросы,
     * которые Feign отправил в Jira stub.
     */
    private fun assertJiraSearchRequests() {
        val requests =
            jiraStub.requests()

        /*
         * Порядок:
         *
         * 1. initiative page 0
         * 2. monitoring Task page 0
         * 3. monitoring Task page 1
         * 4. initiative page 1
         */
        assertThat(requests)
            .hasSize(4)

        val firstInitiativeRequest =
            requests[0]

        assertInitiativeRequest(
            request = firstInitiativeRequest,
            expectedStartAt = 0,
        )

        val firstMonitoringRequest =
            requests[1]

        assertMonitoringRequest(
            request = firstMonitoringRequest,
            expectedStartAt = 0,
        )

        val secondMonitoringRequest =
            requests[2]

        assertMonitoringRequest(
            request = secondMonitoringRequest,
            expectedStartAt = 1,
        )

        val secondInitiativeRequest =
            requests[3]

        assertInitiativeRequest(
            request = secondInitiativeRequest,
            expectedStartAt = 1,
        )
    }

    private fun assertInitiativeRequest(
        request: SearchIssueRequestDto,
        expectedStartAt: Int,
    ) {
        assertThat(request.startAt)
            .isEqualTo(expectedStartAt)

        assertThat(request.maxResults)
            .isEqualTo(PAGE_SIZE)

        assertThat(
            normalizeWhitespace(request.jql)
        )
            .isEqualTo(
                normalizeWhitespace(
                    """
                        project = CROSSGOAL
                        AND issuetype = Инициатива
                        AND resolution = Unresolved
                        AND labels IN (AI_Native_портфель, AI-эффективность)
                        AND created >= -7d
                        ORDER BY created DESC
                    """.trimIndent()
                )
            )

        assertThat(request.fields)
            .containsExactlyElementsOf(
                EXPECTED_INITIATIVE_FIELDS
            )
    }

    private fun assertMonitoringRequest(
        request: SearchIssueRequestDto,
        expectedStartAt: Int,
    ) {
        assertThat(request.startAt)
            .isEqualTo(expectedStartAt)

        assertThat(request.maxResults)
            .isEqualTo(PAGE_SIZE)

        assertThat(
            normalizeWhitespace(request.jql)
        )
            .isEqualTo(
                normalizeWhitespace(
                    """
                        project = CROSSGOAL
                        AND "Epic Link" = $MONITORING_EPIC_KEY
                    """.trimIndent()
                )
            )

        assertThat(request.fields)
            .containsExactlyElementsOf(
                EXPECTED_MONITORING_TASK_FIELDS
            )
    }

    private fun normalizeWhitespace(
        value: String,
    ): String {
        return value
            .trim()
            .replace(
                Regex("\\s+"),
                " ",
            )
    }

    private fun jiraIssueRow(
        resultSet: ResultSet,
    ): JiraIssueRow {
        val rawParentId =
            resultSet.getLong("parent_id")

        val parentId =
            if (resultSet.wasNull()) {
                null
            } else {
                rawParentId
            }

        return JiraIssueRow(
            id =
                resultSet.getLong("id"),
            type =
                resultSet.getString("type"),
            project =
                resultSet.getString("project"),
            parentId =
                parentId,
            jiraId =
                resultSet.getString("jira_id"),
            jiraKey =
                resultSet.getString("jira_key"),
            jiraUrl =
                resultSet.getString("jira_url"),
            qualityGateCode =
                resultSet.getString(
                    "quality_gate_code"
                ),
        )
    }

    companion object {

        /*
         * HTTP server создаётся до поднятия Spring Context.
         */
        private val jiraStub =
            JiraSearchHttpStub()

        /**
         * Перенаправляет реальный Feign
         * с prm-integr на локальный HTTP stub.
         */
        @JvmStatic
        @DynamicPropertySource
        fun dynamicProperties(
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
             * Не допускаем автоматического запуска scheduler
             * во время integration test.
             *
             * "-" — disabled cron для @Scheduled.
             */
            registry.add(
                "scheduled.jira-sync.from-jira-new-cron"
            ) {
                "-"
            }

            registry.add(
                "scheduled.jira-sync.sync-update-cron"
            ) {
                "-"
            }

            registry.add(
                "scheduled.jira-sync.cron"
            ) {
                "-"
            }

            /*
             * В happy-path retry не используется.
             *
             * Если stub неожиданно сломается,
             * тест не должен ждать production delays.
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

        @JvmStatic
        @AfterAll
        fun stopJiraStub() {
            jiraStub.stop()
        }
    }
}

// =============================================================================
// ASSERTION DTO
// =============================================================================

private data class StrategyRelationRow(
    val strategyId: Long,
    val jiraLink: String?,
)

private data class InvolvedResourceRow(
    val source: String,
    val type: String,
    val value: BigDecimal,
    val timeAllocated: String?,
    val created: LocalDateTime?,
    val updated: LocalDateTime?,
)

private data class ContactRow(
    val type: String,
    val email: String,
    val fio: String?,
    val invited: LocalDateTime?,
)

private data class JiraIssueRow(
    val id: Long,
    val type: String,
    val project: String?,
    val parentId: Long?,
    val jiraId: String?,
    val jiraKey: String?,
    val jiraUrl: String?,
    val qualityGateCode: String?,
)

private data class StatusSlaRow(
    val statusCode: String,
    val plannedDate: LocalDateTime?,
    val completedDate: LocalDateTime?,
)
3. Jira HTTP stub

Это можно положить в тот же JiraNewInitiativeImportIntegrationTest.kt ниже класса теста.

Никаких WireMock/MockWebServer зависимостей для него не нужно.

/**
 * Локальный HTTP stub prm-integr/Jira.
 *
 * Обрабатывает реальный endpoint:
 *
 * POST /internal/v1/jira/search
 *
 * Response определяется по JQL + startAt.
 */
private class JiraSearchHttpStub {

    private val objectMapper =
        jacksonObjectMapper()

    private val receivedRequests =
        CopyOnWriteArrayList<
            SearchIssueRequestDto
        >()

    private val executor =
        Executors.newCachedThreadPool { runnable ->
            Thread(
                runnable,
                "jira-fr1-integration-stub",
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
            JIRA_SEARCH_PATH
        ) { exchange ->
            handleSearch(exchange)
        }

        server.executor = executor

        server.start()
    }

    fun baseUrl(): String {
        return "http://localhost:${server.address.port}"
    }

    fun requests():
        List<SearchIssueRequestDto> {

        return receivedRequests.toList()
    }

    fun reset() {
        receivedRequests.clear()
    }

    fun stop() {
        server.stop(0)
        executor.shutdownNow()
    }

    private fun handleSearch(
        exchange: HttpExchange,
    ) {
        try {
            if (
                exchange.requestMethod !=
                HTTP_POST
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
                    isInitiativesRequest(request) ->
                        initiativesResponse(
                            request = request
                        )

                    isMonitoringRequest(request) ->
                        monitoringResponse(
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
                    "Jira integration stub error: " +
                        exception.message,
            )
        }
    }

    private fun isInitiativesRequest(
        request: SearchIssueRequestDto,
    ): Boolean {
        return request.jql.contains(
            "issuetype = Инициатива"
        )
    }

    private fun isMonitoringRequest(
        request: SearchIssueRequestDto,
    ): Boolean {
        return request.jql.contains(
            "\"Epic Link\""
        ) &&
            request.jql.contains(
                MONITORING_EPIC_KEY
            )
    }

    // =========================================================================
    // INITIATIVE SEARCH
    // =========================================================================

    /**
     * Основной Search имеет две страницы:
     *
     * 0 -> новая инициатива
     * 1 -> уже существующая
     */
    private fun initiativesResponse(
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

    private fun newInitiative():
        SearchIssueDto {

        return SearchIssueDto(
            id = NEW_INITIATIVE_JIRA_ID,
            key = NEW_INITIATIVE_KEY,
            fields =
                SearchIssueFieldsDto(
                    summary =
                        NEW_INITIATIVE_SUMMARY,

                    description =
                        LONG_DESCRIPTION,

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
                     * Для smoke test используем реальный
                     * dictionary label.
                     *
                     * Отдельные варианты organization resolver
                     * потом покрываем unit-тестами.
                     */
                    customfield_30001 =
                        listOf(
                            TEST_DIVISION_LABEL
                        ),

                    customfield_34300 =
                        "Оптимизация: 123,45",

                    customfield_30401 =
                        "Доход: 678.90 рублей",

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
                     * Заполнен намеренно.
                     *
                     * Reporter имеет приоритет,
                     * поэтому fallback не должен использоваться.
                     */
                    customfield_29202 =
                        SearchIssueUserInfoDto(
                            emailAddress =
                                FALLBACK_CUSTOMER_EMAIL,
                            displayName =
                                "Fallback customer",
                            active = true,
                        ),

                    /*
                     * Проверяем реальную нормализацию
                     * enabler name.
                     */
                    customfield_15903 =
                        listOf(
                            SearchIssueCheckboxOptionDto(
                                id = 100L,
                                name =
                                    "  GIGA    Chat  ",
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

    private fun existingInitiative():
        SearchIssueDto {

        return SearchIssueDto(
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
    }

    // =========================================================================
    // ISSUE LINKS
    // =========================================================================

    private fun strategyLink():
        SearchIssueLinkDto {

        return SearchIssueLinkDto(
            id = "link-strategy",
            outwardIssue =
                SearchLinkedIssueDto(
                    id = "strategy-jira-id",
                    key = STRATEGY_JIRA_KEY,
                    fields =
                        SearchLinkedIssueFieldsDto(
                            summary =
                                "Стратегия 2026"
                        ),
                ),
        )
    }

    private fun firstGigaUsageLink():
        SearchIssueLinkDto {

        return SearchIssueLinkDto(
            id = "link-giga-1",
            outwardIssue =
                SearchLinkedIssueDto(
                    id = FIRST_GIGAUSAGE_ID,
                    key = FIRST_GIGAUSAGE_KEY,
                ),
        )
    }

    private fun secondGigaUsageLink():
        SearchIssueLinkDto {

        return SearchIssueLinkDto(
            id = "link-giga-2",
            inwardIssue =
                SearchLinkedIssueDto(
                    id = SECOND_GIGAUSAGE_ID,
                    key = SECOND_GIGAUSAGE_KEY,
                ),
        )
    }

    private fun monitoringEpicLink():
        SearchIssueLinkDto {

        return SearchIssueLinkDto(
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
    }

    // =========================================================================
    // MONITORING TASK SEARCH
    // =========================================================================

    /**
     * Monitoring Search также имеет две страницы.
     */
    private fun monitoringResponse(
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
     * Завершённая Task вехи.
     *
     * status.id=10110 ->
     * agent_quality_gate.state=checked.
     */
    private fun completedQualityGateTask():
        SearchIssueDto {

        return SearchIssueDto(
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
    }

    /**
     * Task этапа development.
     *
     * status.id=3 ->
     * текущий agent_status=development.
     */
    private fun developmentTask():
        SearchIssueDto {

        return SearchIssueDto(
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
                        DEVELOPMENT_PLANNED_DATE,

                    resolutiondate = null,
                ),
        )
    }

    // =========================================================================
    // RESPONSE
    // =========================================================================

    private fun sendJsonResponse(
        exchange: HttpExchange,
        response: SearchIssueResponseDto,
    ) {
        val bytes =
            objectMapper.writeValueAsBytes(
                response
            )

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

        exchange.responseBody.use {
            it.write(bytes)
        }
    }

    companion object {
        private const val HTTP_POST =
            "POST"

        private const val JIRA_SEARCH_PATH =
            "/internal/v1/jira/search"
    }
}
4. Константы того же test-файла

Вставь их ниже stub-класса:

private const val NEW_DEPTH =
    7

private const val PAGE_SIZE =
    1

private const val MAX_DESCRIPTION_LENGTH =
    2000

private const val NEW_INITIATIVE_KEY =
    "CROSSGOAL-100"

private const val NEW_INITIATIVE_JIRA_ID =
    "10001"

private const val NEW_INITIATIVE_SUMMARY =
    "Integration FR1 AI-agent"

private val LONG_DESCRIPTION =
    "D".repeat(2100)

private const val EXISTING_INITIATIVE_KEY =
    "CROSSGOAL-200"

private const val EXISTING_INITIATIVE_JIRA_ID =
    "10002"

private const val ANALYSIS_STATUS_CODE =
    "analysis"

private const val DEVELOPMENT_STATUS_CODE =
    "development"

private const val TARGET_SOLUTION_STATUS_CODE =
    "targetSolution"

private const val TEST_BLOCK_CODE =
    "integration-block"

private const val TEST_BLOCK_SHORT_NAME =
    "IB"

private const val TEST_BLOCK_NAME =
    "Integration Block"

private const val TEST_BLOCK_LABEL =
    "Integration Block Label"

private const val TEST_DIVISION_CODE =
    "integration-division"

private const val TEST_DIVISION_SHORT_NAME =
    "ID"

private const val TEST_DIVISION_NAME =
    "Integration Division"

private const val TEST_DIVISION_LABEL =
    "Integration Division Label"

private const val AGENT_INITIATIVE_TYPE =
    "agent"

private const val GEN_AI_INITIATIVE_TYPE =
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

private const val DEVELOPMENT_PLANNED_DATE =
    "2026-09-10T10:00:00.000+0300"

private const val SIGMA_URL_PREFIX =
    "http://jira.test/browse/"

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
