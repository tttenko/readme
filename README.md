```java
      import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.ninjasquad.springmockk.MockkBean
import com.sun.net.httpserver.HttpExchange
import com.sun.net.httpserver.HttpServer
import io.mockk.every
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.jdbc.core.JdbcTemplate
import org.springframework.test.annotation.DirtiesContext
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import java.math.BigDecimal
import java.net.InetSocketAddress
import java.nio.charset.StandardCharsets
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.CopyOnWriteArrayList
import java.util.concurrent.Executors
import java.util.concurrent.atomic.AtomicInteger

/**
 * Сквозная проверка FR2 через настоящий Spring-контекст и HTTP-заглушку prm-integr.
 * Класс размещается рядом с JiraNewInitiativeImportIntegrationTest.
 * Импорты сущностей проекта используются из того же тестового пакета.
 */
@ActiveProfiles("integration-test")
@DirtiesContext(classMode = DirtiesContext.ClassMode.AFTER_CLASS)
class JiraInitiativeUpdateIntegrationTest : AbstractJUnitIntegrationTest() {

    @Autowired private lateinit var updateService: JiraInitiativeUpdateService
    @Autowired private lateinit var agentRepository: AIAgentRepository
    @Autowired private lateinit var optionsRepository: OptionsRepository
    @Autowired private lateinit var statusRepository: StatusRepository
    @Autowired private lateinit var blockRepository: BlockRepository
    @Autowired private lateinit var divisionRepository: DivisionRepository
    @Autowired private lateinit var initiativeTypeRepository: InitiativeTypeRepository
    @Autowired private lateinit var strategyRepository: StrategyRepository
    @Autowired private lateinit var enablerRepository: EnablerRepository
    @Autowired private lateinit var qualityGateRepository: QualityGateRepository
    @Autowired private lateinit var agentQualityGateRepository: AIAgentQualityGateRepository
    @Autowired private lateinit var jiraIssueRepository: JiraIssueRepository
    @Autowired private lateinit var contactRepository: ContactRepository
    @Autowired private lateinit var agentContactRepository: AgentContactRepository
    @Autowired private lateinit var jdbcTemplate: JdbcTemplate

    @MockkBean lateinit var prmAuthAccessTokenService: PrmAuthAccessTokenService

    private lateinit var analysisStatus: StatusEntity
    private lateinit var developmentStatus: StatusEntity
    private lateinit var testDivision: DivisionEntity
    private lateinit var testStrategy: StrategyEntity
    private lateinit var testEnabler: EnablerEntity
    private lateinit var completedGate: QualityGateEntity
    private lateinit var analysisGate: QualityGateEntity
    private lateinit var developmentGate: QualityGateEntity

    @BeforeEach
    fun setUp() {
        clearTables()
        jiraStub.reset()
        every { prmAuthAccessTokenService.getAccessToken() } returns "integration-test-token"
        prepareReferenceData()
    }

    @AfterEach
    fun tearDown() {
        clearTables()
        jiraStub.reset()
    }

    /** Task, инициативы и monitoring обрабатываются в одном запуске с пагинацией. */
    @Test
    fun `should synchronize updated Task and initiative with all related data`() {
        val agent = createAgent(MAIN_KEY)
        seedMonitoring(agent)
        seedContact(agent)
        val previousDescription = agent.agentDescription

        jiraStub.updatedTasks += listOf(qualityGateTask(), developmentTask())
        jiraStub.updatedInitiatives += listOf(initiativeIssue(MAIN_KEY), initiativeIssue(ABSENT_KEY))
        jiraStub.monitoringTasks += listOf(qualityGateTask(), analysisTask(), developmentTask())
        jiraStub.putIssue(MAIN_KEY, "3", "В работе")
        jiraStub.putIssue(ANALYSIS_TASK_KEY, "10109", "To Do")
        jiraStub.putIssue(DEVELOPMENT_TASK_KEY, "3", "In Progress")

        updateService.synchronizeUpdates()

        val updated = requireNotNull(agentRepository.findById(agent.id).orElse(null))
        assertThat(updated.agentName).isEqualTo(UPDATED_SUMMARY)
        assertThat(updated.agentDescription).isEqualTo(previousDescription)
        assertThat(updated.agentStatus?.code).isEqualTo("development")
        assertThat(updated.jiraFromStatus).isEqualTo("done")
        assertThat(updated.jiraUpdated).isNotNull
        assertThat(updated.disabled).isFalse()
        assertThat(updated.block?.id).isEqualTo(testDivision.block?.id)
        assertThat(updated.division?.id).isEqualTo(testDivision.id)
        assertThat(updated.initiativeType?.code).isEqualTo("agent")
        assertThat(updated.agentEffectOptimization).isEqualByComparingTo(BigDecimal("123.45"))
        assertThat(updated.agentEffectRevenue).isEqualByComparingTo(BigDecimal("678.90"))

        assertThat(gateStates(agent.id)[QG_CODE]).isEqualTo("checked")
        assertThat(gateStates(agent.id)[SECOND_QG_CODE]).isEqualTo("unchecked")
        assertThat(stageSla(agent.id, "development").first)
            .isEqualTo(LocalDateTime.of(2026, 9, 10, 10, 0))
        assertThat(stageSla(agent.id, "development").second).isNull()
        assertThat(strategyIds(agent.id)).containsExactly(testStrategy.id)
        assertThat(enablerIds(agent.id)).containsExactly(testEnabler.id)
        assertThat(resourceValue(agent.id)).isEqualByComparingTo(BigDecimal("12.5"))
        assertThat(contactEmails(agent.id)).containsExactly("original@sber.ru")

        assertThat(jiraKeys(agent.id, "initiative", "gigausage"))
            .containsExactlyInAnyOrder("GIGAUSAGE-100", "GIGAUSAGE-200")
        val epic = jiraIssueRepository.findByAgentIdAndTypeAndProject(agent.id, "epic", "crossgoal").single()
        assertThat(epic.jiraKey).isEqualTo(NEW_EPIC_KEY)
        val savedTasks = jiraIssueRepository.findAllByAgentIdAndType(agent.id, "task")
        assertThat(savedTasks.map { it.jiraKey })
            .containsExactlyInAnyOrder(QG_TASK_KEY, ANALYSIS_TASK_KEY, DEVELOPMENT_TASK_KEY)
        assertThat(savedTasks.map { it.parentId }.distinct()).containsExactly(epic.id)
        assertThat(agentRepository.count()).isEqualTo(1L) // ABSENT_KEY не создаётся.

        assertThat(jiraStub.searchRequests(Fr2SearchKind.UPDATED_TASKS).map { it.startAt }).containsExactly(0, 1)
        assertThat(jiraStub.searchRequests(Fr2SearchKind.INITIATIVES).map { it.startAt }).containsExactly(0, 1)
        assertThat(jiraStub.searchRequests(Fr2SearchKind.MONITORING).map { it.startAt }).containsExactly(0, 1, 2)
        assertThat(jiraStub.issueRequests()).contains(MAIN_KEY, ANALYSIS_TASK_KEY, DEVELOPMENT_TASK_KEY)
        assertSearchContract()
    }

    /** Пустые результаты не создают и не изменяют инициативы. */
    @Test
    fun `should finish when both searches return total zero`() {
        val agent = createAgent(MAIN_KEY)
        val originalUpdated = requireNotNull(agent.updated)

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(agent.id).orElseThrow().updated).isEqualTo(originalUpdated)
        assertThat(jiraStub.searchRequests(Fr2SearchKind.UPDATED_TASKS)).hasSize(1)
        assertThat(jiraStub.searchRequests(Fr2SearchKind.INITIATIVES)).hasSize(1)
        assertThat(jiraStub.issueRequests()).isEmpty()
    }

    /** Task без связи в jira_issue пропускается без GET инициативы. */
    @Test
    fun `should skip Jira Task absent in Pult`() {
        val agent = createAgent(MAIN_KEY)
        jiraStub.updatedTasks += qualityGateTask()

        updateService.synchronizeUpdates()

        assertThat(gateStates(agent.id)[QG_CODE]).isEqualTo("unchecked")
        assertThat(jiraStub.issueRequests()).isEmpty()
        assertThat(jiraStub.searchRequests(Fr2SearchKind.INITIATIVES)).hasSize(1)
    }

    /** Повторный GET всех этапов определяет максимальный InProgress, затем минимальный To Do. */
    @Test
    fun `should recalculate stage status from fresh Jira GET responses`() {
        val agent = createAgent(MAIN_KEY)
        seedMonitoring(agent)
        jiraStub.updatedTasks += developmentTask()
        jiraStub.putIssue(MAIN_KEY, "3", "В работе")
        jiraStub.putIssue(ANALYSIS_TASK_KEY, "3", "In Progress")
        jiraStub.putIssue(DEVELOPMENT_TASK_KEY, "3", "In Progress")

        updateService.synchronizeUpdates()
        assertThat(agentRepository.findById(agent.id).orElseThrow().agentStatus?.code).isEqualTo("development")

        jiraStub.reset()
        jiraStub.updatedTasks += developmentTask()
        jiraStub.putIssue(MAIN_KEY, "3", "В работе")
        jiraStub.putIssue(ANALYSIS_TASK_KEY, "10109", "To Do")
        jiraStub.putIssue(DEVELOPMENT_TASK_KEY, "4", "Reopened")

        updateService.synchronizeUpdates()
        assertThat(agentRepository.findById(agent.id).orElseThrow().agentStatus?.code).isEqualTo("analysis")

        jiraStub.reset()
        jiraStub.updatedTasks += developmentTask()
        jiraStub.putIssue(MAIN_KEY, "3", "В работе")
        jiraStub.putIssue(ANALYSIS_TASK_KEY, "10110", "Done")
        jiraStub.putIssue(DEVELOPMENT_TASK_KEY, "10110", "Done")

        updateService.synchronizeUpdates()
        assertThat(agentRepository.findById(agent.id).orElseThrow().agentStatus?.code).isEqualTo("targetSolution")
    }

    /** Уточнение 7.b.iii: статус родительской инициативы проверяется при обработке Task. */
    @Test
    fun `should archive parent initiative cancelled in Jira before updating Task`() {
        val agent = createAgent(MAIN_KEY)
        seedMonitoring(agent)
        jiraStub.updatedTasks += qualityGateTask()
        jiraStub.putIssue(MAIN_KEY, "999", "Отменена")

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(agent.id).orElseThrow().disabled).isTrue()
        assertThat(gateStates(agent.id)[QG_CODE]).isEqualTo("unchecked")
        assertThat(jiraStub.issueRequests()).containsExactly(MAIN_KEY)
    }

    /** Та же проверка родителя архивирует инициативу с меткой ClassicML. */
    @Test
    fun `should archive parent initiative with ClassicML label before updating Task`() {
        val agent = createAgent(MAIN_KEY)
        seedMonitoring(agent)
        jiraStub.updatedTasks += qualityGateTask()
        jiraStub.putIssue(MAIN_KEY, "3", "В работе", labels = listOf("ClassicML"))

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(agent.id).orElseThrow().disabled).isTrue()
        assertThat(gateStates(agent.id)[QG_CODE]).isEqualTo("unchecked")
    }

    /** Незавершённая Jira Task снимает ранее установленную отметку вехи. */
    @Test
    fun `should change checked quality gate to unchecked`() {
        val agent = createAgent(MAIN_KEY)
        seedMonitoring(agent)
        agentQualityGateRepository.upsertStateForAgent(agent.id, listOf(QG_CODE), "checked", LocalDateTime.now())
        jiraStub.updatedTasks += qualityGateTask().copy(fields = SearchIssueFieldsDto(
            summary = "Архитектура: согласование", status = SearchIssueStatusDto(id = "10109", name = "To Do")
        ))
        jiraStub.putIssue(MAIN_KEY, "3", "В работе")

        updateService.synchronizeUpdates()

        assertThat(gateStates(agent.id)[QG_CODE]).isEqualTo("unchecked")
        assertThat(jiraStub.issueRequests()).containsExactly(MAIN_KEY)
    }

    /** Обновление Task сохраняет фактическую дату завершения этапа. */
    @Test
    fun `should save planned and completed dates of changed stage`() {
        val agent = createAgent(MAIN_KEY)
        seedMonitoring(agent)
        jiraStub.updatedTasks += developmentTask().copy(fields = SearchIssueFieldsDto(
            summary = "Этап: разработка", status = SearchIssueStatusDto(id = "10110", name = "Done"),
            customfield_16701 = "2026-09-10T10:00:00.000+0300",
            resolutiondate = "2026-09-11T12:00:00.000+0300"
        ))
        jiraStub.putIssue(MAIN_KEY, "3", "В работе")
        jiraStub.putIssue(ANALYSIS_TASK_KEY, "10109", "To Do")
        jiraStub.putIssue(DEVELOPMENT_TASK_KEY, "10110", "Done")

        updateService.synchronizeUpdates()

        assertThat(stageSla(agent.id, "development"))
            .isEqualTo(LocalDateTime.of(2026, 9, 10, 10, 0) to LocalDateTime.of(2026, 9, 11, 12, 0))
        assertThat(agentRepository.findById(agent.id).orElseThrow().agentStatus?.code).isEqualTo("analysis")
    }

    /** Уточнения 7.b.iii и 10.c.iii: архивируем найденную по ключу инициативу. */
    @Test
    fun `should archive cancelled and ClassicML initiatives found by Search`() {
        val cancelledAgent = createAgent(MAIN_KEY)
        val classicAgent = createAgent(SECOND_KEY)
        jiraStub.updatedInitiatives += listOf(
            initiativeIssue(MAIN_KEY, statusName = "Отменена", labels = emptyList()),
            initiativeIssue(SECOND_KEY, labels = listOf("ClassicML"))
        )

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(cancelledAgent.id).orElseThrow().disabled).isTrue()
        assertThat(agentRepository.findById(classicAgent.id).orElseThrow().disabled).isTrue()
        assertThat(jiraStub.searchRequests(Fr2SearchKind.MONITORING)).isEmpty()
    }

    /** Обновление в первый час после создания и более новая версия Пульта не применяются. */
    @Test
    fun `should skip initiative younger than one hour and initiative updated later in Pult`() {
        val youngAgent = createAgent(MAIN_KEY)
        val pultAgent = createAgent(SECOND_KEY, pultUpdated = LocalDateTime.now().plusHours(2))
        jiraStub.updatedInitiatives += listOf(
            initiativeIssue(MAIN_KEY, createdAt = LocalDateTime.now().minusMinutes(50),
                updatedAt = LocalDateTime.now().minusMinutes(20)),
            initiativeIssue(SECOND_KEY)
        )

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(youngAgent.id).orElseThrow().agentName).isEqualTo("Previous name")
        assertThat(agentRepository.findById(pultAgent.id).orElseThrow().agentName).isEqualTo("Previous name")
        assertThat(jiraStub.searchRequests(Fr2SearchKind.MONITORING)).isEmpty()
    }

    /** Инициатива находится и по agent_jira_url, если jira_issue для неё отсутствует. */
    @Test
    fun `should find existing initiative by agent Jira URL`() {
        val agent = createAgent("PULT-101", jiraKey = MAIN_KEY, createInitiativeIssue = false)
        jiraStub.updatedInitiatives += initiativeIssue(MAIN_KEY, includeMonitoringLink = false)

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(agent.id).orElseThrow().agentName).isEqualTo(UPDATED_SUMMARY)
        assertThat(agentRepository.findById(agent.id).orElseThrow().jiraFromStatus).isEqualTo("done")
        assertThat(jiraIssueRepository.findByAgentIdAndTypeAndProject(agent.id, "initiative", "crossgoal")).isEmpty()
    }

    /** Без корректных created/updated сравнение дат невозможно; текущие поля сохраняются. */
    @Test
    fun `should skip initiative with invalid Jira date`() {
        val agent = createAgent(MAIN_KEY)
        jiraStub.updatedInitiatives += initiativeIssue(MAIN_KEY).copy(fields =
            initiativeIssue(MAIN_KEY).fields?.copy(updated = "not-a-date"))

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(agent.id).orElseThrow().agentName).isEqualTo("Previous name")
        assertThat(jiraStub.searchRequests(Fr2SearchKind.MONITORING)).isEmpty()
    }

    /** PULT сохраняет бизнес-статус, не перечитывает Epic/Task; стратегии и ресурсы обновляются. */
    @Test
    fun `should update PULT initiative without changing monitoring or business status`() {
        val agent = createAgent("PULT-100", jiraKey = MAIN_KEY)
        jiraStub.updatedInitiatives += initiativeIssue(MAIN_KEY)

        updateService.synchronizeUpdates()

        val updated = agentRepository.findById(agent.id).orElseThrow()
        assertThat(updated.agentName).isEqualTo(UPDATED_SUMMARY)
        assertThat(updated.agentStatus?.code).isEqualTo("analysis")
        assertThat(updated.jiraFromStatus).isEqualTo("done")
        assertThat(strategyIds(agent.id)).containsExactly(testStrategy.id)
        assertThat(resourceValue(agent.id)).isEqualByComparingTo(BigDecimal("12.5"))
        assertThat(enablerIds(agent.id)).isEmpty()
        assertThat(jiraKeys(agent.id, "initiative", "gigausage")).isEmpty()
        assertThat(jiraStub.searchRequests(Fr2SearchKind.MONITORING)).isEmpty()
        assertThat(jiraStub.issueRequests()).isEmpty()
    }

    /** Для AI-эффективности Epic не требуется. */
    @Test
    fun `should finish AI effectiveness initiative in analysis without monitoring`() {
        val agent = createAgent(MAIN_KEY)
        jiraStub.updatedInitiatives += initiativeIssue(MAIN_KEY, labels = listOf("AI-эффективность"))

        updateService.synchronizeUpdates()

        val updated = agentRepository.findById(agent.id).orElseThrow()
        assertThat(updated.agentStatus?.code).isEqualTo("analysis")
        assertThat(updated.jiraFromStatus).isEqualTo("done")
        assertThat(jiraStub.searchRequests(Fr2SearchKind.MONITORING)).isEmpty()
    }

    /** Отсутствие monitoring Epic завершает синхронизацию со статусом analysis. */
    @Test
    fun `should finish initiative without monitoring Epic in analysis`() {
        val agent = createAgent(MAIN_KEY)
        jiraStub.updatedInitiatives += initiativeIssue(MAIN_KEY, includeMonitoringLink = false)

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(agent.id).orElseThrow().agentStatus?.code).isEqualTo("analysis")
        assertThat(agentRepository.findById(agent.id).orElseThrow().jiraFromStatus).isEqualTo("done")
        assertThat(jiraStub.searchRequests(Fr2SearchKind.MONITORING)).isEmpty()
    }

    /** После ошибки monitoring Search инициатива error, при следующем запуске может завершиться. */
    @Test
    fun `should retry incomplete synchronization on next scheduler run`() {
        val agent = createAgent(MAIN_KEY)
        jiraStub.updatedInitiatives += initiativeIssue(MAIN_KEY)
        jiraStub.failSearch(Fr2SearchKind.MONITORING)

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(agent.id).orElseThrow().jiraFromStatus).isEqualTo("error")
        assertThat(jiraIssueRepository.findByAgentIdAndTypeAndProject(agent.id, "epic", "crossgoal")).isEmpty()
        assertThat(jiraStub.searchRequests(Fr2SearchKind.MONITORING)).hasSize(3)

        jiraStub.reset()
        jiraStub.updatedInitiatives += initiativeIssue(MAIN_KEY)
        jiraStub.monitoringTasks += listOf(qualityGateTask(), developmentTask())

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(agent.id).orElseThrow().jiraFromStatus).isEqualTo("done")
        assertThat(agentRepository.findById(agent.id).orElseThrow().agentStatus?.code).isEqualTo("development")
        assertThat(jiraIssueRepository.findByAgentIdAndTypeAndProject(agent.id, "epic", "crossgoal")).hasSize(1)
    }

    /** Невозможно присвоить targetSolution, если мониторинг вернул пустой набор этапов. */
    @Test
    fun `should mark synchronization error when monitoring contains no stage Task`() {
        val agent = createAgent(MAIN_KEY)
        jiraStub.updatedInitiatives += initiativeIssue(MAIN_KEY)
        jiraStub.monitoringTasks += qualityGateTask()

        updateService.synchronizeUpdates()

        assertThat(agentRepository.findById(agent.id).orElseThrow().jiraFromStatus).isEqualTo("error")
        assertThat(jiraIssueRepository.findByAgentIdAndTypeAndProject(agent.id, "epic", "crossgoal")).isEmpty()
    }

    /** Окончательная ошибка первого Search останавливает оба подпроцесса. */
    @Test
    fun `should stop scheduler after exhausted updated Task Search retries`() {
        val agent = createAgent(MAIN_KEY)
        jiraStub.updatedInitiatives += initiativeIssue(MAIN_KEY)
        jiraStub.failSearch(Fr2SearchKind.UPDATED_TASKS)

        updateService.synchronizeUpdates()

        assertThat(jiraStub.searchRequests(Fr2SearchKind.UPDATED_TASKS)).hasSize(3)
        assertThat(jiraStub.searchRequests(Fr2SearchKind.INITIATIVES)).isEmpty()
        assertThat(agentRepository.findById(agent.id).orElseThrow().agentName).isEqualTo("Previous name")
    }

    /** Временная HTTP-ошибка GET устраняется имеющимся Feign retry. */
    @Test
    fun `should recover when parent Jira GET succeeds on third attempt`() {
        val agent = createAgent(MAIN_KEY)
        seedMonitoring(agent)
        jiraStub.updatedTasks += qualityGateTask()
        jiraStub.putIssue(MAIN_KEY, "3", "В работе")
        jiraStub.failGet(MAIN_KEY, 2)

        updateService.synchronizeUpdates()

        assertThat(jiraStub.issueRequests().count { it == MAIN_KEY }).isEqualTo(3)
        assertThat(gateStates(agent.id)[QG_CODE]).isEqualTo("checked")
        assertThat(agentRepository.findById(agent.id).orElseThrow().jiraFromStatus).isEqualTo("done")
        assertThat(jiraStub.searchRequests(Fr2SearchKind.INITIATIVES)).hasSize(1)
    }

    /** Успешный Task-подпроцесс не скрывает окончательную ошибку Search инициатив. */
    @Test
    fun `should stop initiative subprocess after exhausted Jira Search retries`() {
        val agent = createAgent(MAIN_KEY)
        jiraStub.updatedInitiatives += initiativeIssue(MAIN_KEY)
        jiraStub.failSearch(Fr2SearchKind.INITIATIVES)

        updateService.synchronizeUpdates()

        assertThat(jiraStub.searchRequests(Fr2SearchKind.UPDATED_TASKS)).hasSize(1)
        assertThat(jiraStub.searchRequests(Fr2SearchKind.INITIATIVES)).hasSize(3)
        assertThat(agentRepository.findById(agent.id).orElseThrow().agentName).isEqualTo("Previous name")
    }

    /** Окончательная ошибка GET одного этапа не блокирует следующую инициативу. */
    @Test
    fun `should mark one agent error and continue with another after Jira GET failure`() {
        val failing = createAgent(MAIN_KEY)
        val succeeding = createAgent(SECOND_KEY, jiraKey = SECOND_KEY)
        seedMonitoring(failing)
        seedMonitoring(succeeding, qualityGateTaskKey = "CROSSGOAL-601",
            analysisTaskKey = "CROSSGOAL-602", developmentTaskKey = "CROSSGOAL-603")
        jiraStub.updatedTasks += listOf(developmentTask(), developmentTask("CROSSGOAL-603"))
        jiraStub.putIssue(MAIN_KEY, "3", "В работе")
        jiraStub.putIssue(SECOND_KEY, "3", "В работе")
        jiraStub.failGet(ANALYSIS_TASK_KEY, 10)
        jiraStub.putIssue(DEVELOPMENT_TASK_KEY, "3", "In Progress")
        jiraStub.putIssue("CROSSGOAL-602", "10109", "To Do")
        jiraStub.putIssue("CROSSGOAL-603", "3", "In Progress")

        updateService.synchronizeUpdates()

        assertThat(jiraStub.issueRequests().count { it == ANALYSIS_TASK_KEY }).isEqualTo(3)
        assertThat(agentRepository.findById(failing.id).orElseThrow().jiraFromStatus).isEqualTo("error")
        assertThat(agentRepository.findById(succeeding.id).orElseThrow().jiraFromStatus).isEqualTo("done")
        assertThat(agentRepository.findById(succeeding.id).orElseThrow().agentStatus?.code).isEqualTo("development")
        assertThat(jiraStub.searchRequests(Fr2SearchKind.INITIATIVES)).hasSize(1)
    }

    /** Лимит превышается на 16-й окончательной ошибке; 17-я инициатива не обрабатывается. */
    @Test
    fun `should stop after more than fifteen terminal Jira GET failures`() {
        val failingStageKeys = mutableListOf<String>()
        repeat(17) { index ->
            val agentKey = "CROSSGOAL-${1000 + index}"
            val qualityGateKey = "CROSSGOAL-${2000 + index * 3}"
            val analysisKey = "CROSSGOAL-${2001 + index * 3}"
            val developmentKey = "CROSSGOAL-${2002 + index * 3}"
            val agent = createAgent(agentKey)
            seedMonitoring(agent, qualityGateKey, analysisKey, developmentKey)
            jiraStub.updatedTasks += developmentTask(developmentKey)
            jiraStub.putIssue(agentKey, "3", "В работе")
            jiraStub.failGet(analysisKey, 10)
            failingStageKeys += analysisKey
        }

        updateService.synchronizeUpdates()

        assertThat(failingStageKeys.take(16)).allSatisfy { stageKey ->
            assertThat(jiraStub.issueRequests().count { it == stageKey }).isEqualTo(3)
        }
        assertThat(jiraStub.issueRequests()).doesNotContain(failingStageKeys.last())
        assertThat(jiraStub.searchRequests(Fr2SearchKind.INITIATIVES)).isEmpty()
    }

    private fun prepareReferenceData() {
        val options = optionsRepository.findFirstByOrderByIdAsc() ?: OptionsEntity()
        options.newDepth = 7
        options.updateDepth = 3
        options.maxResults = PAGE_SIZE
        optionsRepository.saveAndFlush(options)

        analysisStatus = requireNotNull(statusRepository.findFirstByCode("analysis"))
        developmentStatus = requireNotNull(statusRepository.findFirstByCode("development"))
        requireNotNull(statusRepository.findFirstByCode("targetSolution"))

        val block = blockRepository.saveAndFlush(BlockEntity().apply {
            code = "integration-fr2-block"
            shortName = "FR2"
            name = "FR2 Block"
            label = "FR2 Block Label"
            disabled = false
        })
        testDivision = divisionRepository.saveAndFlush(DivisionEntity().apply {
            this.block = block
            code = "integration-fr2-division"
            shortName = "FR2D"
            name = "FR2 Division"
            label = "FR2 Division Label"
            ordering = 10L
            disabled = false
        })
        initiativeTypeRepository.saveAndFlush(InitiativeTypeEntity("agent", "AI-агент", "Agent"))
        initiativeTypeRepository.saveAndFlush(InitiativeTypeEntity("genAiSolution", "GenAI", "GenAI"))
        testStrategy = strategyRepository.saveAndFlush(StrategyEntity(jiraIssue = "STRATEGY-2026", name = "Strategy FR2"))
        testEnabler = enablerRepository.saveAndFlush(EnablerEntity().apply {
            name = "Giga Chat"
            shortDescription = "FR2"
            description = "FR2 enabler"
            disabled = false
        })
        completedGate = qualityGateRepository.saveAndFlush(QualityGateEntity(
            code = QG_CODE, name = "Архитектурное согласование", type = QualityGateType.quality_gate,
            status = analysisStatus, ordering = 9100, regexp = "архитектура,согласование", disabled = false
        ))
        qualityGateRepository.saveAndFlush(QualityGateEntity(
            code = SECOND_QG_CODE, name = "Безопасность", type = QualityGateType.quality_gate,
            status = analysisStatus, ordering = 9200, regexp = "безопасность", disabled = false
        ))
        analysisGate = qualityGateRepository.saveAndFlush(QualityGateEntity(
            code = "STAGE_ANALYSIS_FR2", name = "Концепция", type = QualityGateType.status,
            status = analysisStatus, ordering = 9250, regexp = "этап,концепция", disabled = false
        ))
        developmentGate = qualityGateRepository.saveAndFlush(QualityGateEntity(
            code = "STAGE_DEVELOPMENT_FR2", name = "Разработка", type = QualityGateType.status,
            status = developmentStatus, ordering = 9300, regexp = "этап,разработка", disabled = false
        ))
    }

    private fun createAgent(
        agentId: String, jiraKey: String = agentId,
        pultUpdated: LocalDateTime = LocalDateTime.now().minusDays(3),
        createInitiativeIssue: Boolean = true,
    ): AIAgentEntity {
        val agent = agentRepository.saveAndFlush(AIAgentEntity(
            agentId = agentId, agentName = "Previous name", agentDescription = "Description from Pult",
            agentJiraUrl = jiraKey, agentStatus = analysisStatus
        ).apply {
            created = LocalDateTime.now().minusDays(5)
            updated = pultUpdated
            jiraFromStatus = "done"
            disabled = false
        })
        agentQualityGateRepository.insertUncheckedForAgent(agent.id, listOf(QG_CODE, SECOND_QG_CODE), LocalDateTime.now())
        if (createInitiativeIssue) {
            jiraIssueRepository.saveAndFlush(JiraIssueEntity(
                agent = agent, type = "initiative", project = "crossgoal", jiraKey = jiraKey,
                jiraId = jiraKey.substringAfter('-'), jiraUrl = "$SIGMA_URL_PREFIX$jiraKey"
            ))
        }
        return agent
    }

    private fun seedMonitoring(
        agent: AIAgentEntity,
        qualityGateTaskKey: String = QG_TASK_KEY,
        analysisTaskKey: String = ANALYSIS_TASK_KEY,
        developmentTaskKey: String = DEVELOPMENT_TASK_KEY,
    ) {
        val epic = jiraIssueRepository.saveAndFlush(JiraIssueEntity(
            agent = agent, type = "epic", project = "crossgoal", jiraKey = "CROSSGOAL-${4000 + agent.id}"
        ))
        listOf(
            qualityGateTaskKey to completedGate,
            analysisTaskKey to analysisGate,
            developmentTaskKey to developmentGate,
            "CROSSGOAL-${5000 + agent.id}" to completedGate // Старая Task должна удалиться.
        ).forEach { (taskKey, gate) ->
            jiraIssueRepository.saveAndFlush(JiraIssueEntity(
                agent = agent, type = "task", project = "crossgoal", parentId = epic.id,
                jiraKey = taskKey, qualityGate = gate
            ))
        }
        jiraIssueRepository.saveAndFlush(JiraIssueEntity(
            agent = agent, type = "initiative", project = "gigausage", jiraKey = "GIGAUSAGE-${900 + agent.id}"
        ))
    }

    private fun seedContact(agent: AIAgentEntity) {
        val contact = contactRepository.saveAndFlush(ContactEntity(email = "original@sber.ru", fio = "Original Contact", invited = null))
        agentContactRepository.saveAndFlush(AgentContactEntity(agent = agent, type = "customer", contact = contact, userId = null))
    }

    private fun initiativeIssue(
        key: String,
        statusName: String = "В работе",
        labels: List<String> = listOf("AI_Native_портфель", "AI-агент"),
        includeMonitoringLink: Boolean = true,
        createdAt: LocalDateTime = LocalDateTime.now().minusDays(2),
        updatedAt: LocalDateTime = LocalDateTime.now().minusHours(1),
    ): SearchIssueDto {
        val links = listOf(
            SearchIssueLinkDto(outwardIssue = SearchLinkedIssueDto(key = "STRATEGY-2026",
                fields = SearchLinkedIssueFieldsDto(summary = "Стратегия 2026"))),
            SearchIssueLinkDto(outwardIssue = SearchLinkedIssueDto(id = "9100", key = "GIGAUSAGE-100")),
            SearchIssueLinkDto(inwardIssue = SearchLinkedIssueDto(id = "9200", key = "GIGAUSAGE-200"))
        ) + if (includeMonitoringLink) listOf(SearchIssueLinkDto(outwardIssue = SearchLinkedIssueDto(
            id = "50000", key = NEW_EPIC_KEY,
            fields = SearchLinkedIssueFieldsDto(summary = "Мониторинг портфеля AI-Native")
        ))) else emptyList()

        return SearchIssueDto(id = key.substringAfter('-'), key = key, fields = SearchIssueFieldsDto(
            summary = UPDATED_SUMMARY, description = "Jira description must not replace Pult description",
            status = SearchIssueStatusDto(id = "3", name = statusName), labels = labels,
            customfield_30001 = listOf("FR2 Division Label"),
            customfield_34300 = "123,45", customfield_30401 = "678.90",
            customfield_31304 = "12,5 FTE",
            customfield_15903 = listOf(SearchIssueCheckboxOptionDto(name = "  GIGA    Chat ", checked = true)),
            assignee = SearchIssueUserInfoDto(emailAddress = "jira-developer@sber.ru"),
            reporter = SearchIssueUserInfoDto(emailAddress = "jira-customer@sber.ru"),
            issuelinks = links, created = jiraDate(createdAt), updated = jiraDate(updatedAt)
        ))
    }

    private fun qualityGateTask(key: String = QG_TASK_KEY) = SearchIssueDto(id = key.substringAfter('-'), key = key,
        fields = SearchIssueFieldsDto(summary = "Архитектура: согласование",
            status = SearchIssueStatusDto(id = "10110", name = "Done")))

    private fun analysisTask(key: String = ANALYSIS_TASK_KEY) = SearchIssueDto(id = key.substringAfter('-'), key = key,
        fields = SearchIssueFieldsDto(summary = "Этап: концепция",
            status = SearchIssueStatusDto(id = "10109", name = "To Do")))

    private fun developmentTask(key: String = DEVELOPMENT_TASK_KEY) = SearchIssueDto(id = key.substringAfter('-'), key = key,
        fields = SearchIssueFieldsDto(summary = "Этап: разработка",
            status = SearchIssueStatusDto(id = "3", name = "In Progress"),
            customfield_16701 = "2026-09-10T10:00:00.000+0300"))

    private fun gateStates(agentId: Long): Map<String, String> = jdbcTemplate.query(
        "select quality_gate_code, state from agent_quality_gate where ai_agent_id = ?",
        { resultSet, _ -> resultSet.getString("quality_gate_code") to resultSet.getString("state") }, agentId
    ).toMap()

    private fun stageSla(agentId: Long, statusCode: String): Pair<LocalDateTime?, LocalDateTime?> {
        val rows = jdbcTemplate.query("""
            select sla.planned_date, sla.completed_date from agent_status_sla sla
            join status s on s.id = sla.agent_status_id
            where sla.ai_agent_id = ? and s.code = ?
        """.trimIndent(), { resultSet, _ ->
            resultSet.getTimestamp("planned_date")?.toLocalDateTime() to
                resultSet.getTimestamp("completed_date")?.toLocalDateTime()
        }, agentId, statusCode)
        assertThat(rows).hasSize(1)
        return rows.single()
    }

    private fun strategyIds(agentId: Long): List<Long> = jdbcTemplate.queryForList(
        "select strategy_id from agent_strategy where ai_agent_id = ?", Long::class.java, agentId
    )

    private fun enablerIds(agentId: Long): List<Long> = jdbcTemplate.queryForList(
        "select enabler_id from agent_enabler where agent_id = ?", Long::class.java, agentId
    )

    private fun resourceValue(agentId: Long): BigDecimal? = jdbcTemplate.queryForObject(
        "select value from involved_resource where ai_agent_id = ? and source = 'without_steerCo' and type = 'business'",
        BigDecimal::class.java, agentId
    )

    private fun contactEmails(agentId: Long): List<String> = jdbcTemplate.queryForList("""
        select contact.email from agent_contact relation
        join contact contact on contact.id = relation.contact_id
        where relation.agent_id = ?
    """.trimIndent(), String::class.java, agentId)

    private fun jiraKeys(agentId: Long, type: String, project: String): List<String?> =
        jiraIssueRepository.findByAgentIdAndTypeAndProject(agentId, type, project).map { it.jiraKey }

    private fun assertSearchContract() {
        val taskRequest = jiraStub.searchRequests(Fr2SearchKind.UPDATED_TASKS).first()
        val initiativeRequest = jiraStub.searchRequests(Fr2SearchKind.INITIATIVES).first()
        assertThat(taskRequest.maxResults).isEqualTo(PAGE_SIZE)
        assertThat(taskRequest.jql).contains("project = CROSSGOAL", "issuetype = Task",
            "updated >= -3d", "ORDER BY updated DESC", "\"Epic Link\"=\"Мониторинг портфеля AI-Native\"")
        assertThat(taskRequest.fields).contains("status", "customfield_16701", "resolutiondate")
        assertThat(initiativeRequest.maxResults).isEqualTo(PAGE_SIZE)
        assertThat(initiativeRequest.jql).contains("issuetype = Инициатива", "labels = ClassicML",
            "status = \"Отменена\"", "updated >= -3d", "ORDER BY updated DESC")
        assertThat(initiativeRequest.fields).contains("summary", "status", "labels", "issuelinks", "updated", "created")
    }

    companion object {
        private val jiraStub = Fr2JiraHttpStub()

        @JvmStatic
        @DynamicPropertySource
        fun dynamicProperties(registry: DynamicPropertyRegistry) {
            registry.add("rest.integr.base-url") { jiraStub.baseUrl() }
            registry.add("scheduled.jira-sync.sigma.url.prefix") { SIGMA_URL_PREFIX }
            registry.add("scheduled.jira-sync.delta.url.prefix") { "http://jira.test/delta/" }
            registry.add("scheduled.jira-sync.from-jira-update-cron") { "-" }
            registry.add("scheduled.jira-sync.from-jira-new-cron") { "-" }
            registry.add("scheduled.jira-sync.sync-update-cron") { "-" }
            registry.add("scheduled.jira-sync.cron") { "-" }
            registry.add("retry.maxAttempts") { "3" }
            registry.add("retry.delay") { "1" }
            registry.add("retry.multiplier") { "1.0" }
        }

        @JvmStatic @AfterAll
        fun stopJiraStub() = jiraStub.stop()
    }
}

private enum class Fr2SearchKind { UPDATED_TASKS, INITIATIVES, MONITORING }

/** Реальный HTTP endpoint prm-integr; ответы задаются сценарием каждого теста. */
private class Fr2JiraHttpStub {
    val updatedTasks = CopyOnWriteArrayList<SearchIssueDto>()
    val updatedInitiatives = CopyOnWriteArrayList<SearchIssueDto>()
    val monitoringTasks = CopyOnWriteArrayList<SearchIssueDto>()

    private val objectMapper = jacksonObjectMapper()
    private val searchCalls = CopyOnWriteArrayList<Pair<Fr2SearchKind, SearchIssueRequestDto>>()
    private val issueCalls = CopyOnWriteArrayList<String>()
    private val issues = ConcurrentHashMap<String, Map<String, Any>>()
    private val failingSearches = ConcurrentHashMap.newKeySet<Fr2SearchKind>()
    private val getFailuresRemaining = ConcurrentHashMap<String, AtomicInteger>()
    private val executor = Executors.newCachedThreadPool { runnable ->
        Thread(runnable, "jira-fr2-integration-stub").apply { isDaemon = true }
    }
    private val server = HttpServer.create(InetSocketAddress("localhost", 0), 0)

    init {
        server.createContext("/internal/v1/jira/search") { exchange -> handleSearch(exchange) }
        server.createContext("/internal/v1/jira/issue/") { exchange -> handleGet(exchange) }
        server.executor = executor
        server.start()
    }

    fun baseUrl() = "http://localhost:${server.address.port}"
    fun searchRequests(kind: Fr2SearchKind) = searchCalls.filter { it.first == kind }.map { it.second }
    fun issueRequests() = issueCalls.toList()
    fun failSearch(kind: Fr2SearchKind) { failingSearches += kind }
    fun failGet(issueKey: String, attempts: Int) { getFailuresRemaining[issueKey] = AtomicInteger(attempts) }

    fun putIssue(issueKey: String, statusId: String, statusName: String, labels: List<String> = emptyList()) {
        issues[issueKey] = mapOf("id" to issueKey.substringAfter('-'), "key" to issueKey,
            "fields" to mapOf("status" to mapOf("id" to statusId, "name" to statusName), "labels" to labels))
    }

    fun reset() {
        updatedTasks.clear()
        updatedInitiatives.clear()
        monitoringTasks.clear()
        searchCalls.clear()
        issueCalls.clear()
        issues.clear()
        failingSearches.clear()
        getFailuresRemaining.clear()
    }

    fun stop() {
        server.stop(0)
        executor.shutdownNow()
    }

    private fun handleSearch(exchange: HttpExchange) {
        try {
            if (exchange.requestMethod != "POST") return respond(exchange, 405, "Only POST")
            val request = objectMapper.readValue(exchange.requestBody, SearchIssueRequestDto::class.java)
            val kind = when {
                request.jql.contains("issuetype = Инициатива") -> Fr2SearchKind.INITIATIVES
                request.jql.contains("Мониторинг портфеля AI-Native") -> Fr2SearchKind.UPDATED_TASKS
                request.jql.contains("\"Epic Link\"") -> Fr2SearchKind.MONITORING
                else -> return respond(exchange, 500, "Unexpected JQL: ${request.jql}")
            }
            searchCalls += kind to request
            if (kind in failingSearches) return respond(exchange, 500, "Jira Search failed")
            val allIssues = when (kind) {
                Fr2SearchKind.UPDATED_TASKS -> updatedTasks
                Fr2SearchKind.INITIATIVES -> updatedInitiatives
                Fr2SearchKind.MONITORING -> monitoringTasks
            }
            val response = SearchIssueResponseDto(startAt = request.startAt, maxResults = request.maxResults,
                total = allIssues.size, issues = allIssues.drop(request.startAt ?: 0).take(request.maxResults))
            respond(exchange, 200, objectMapper.writeValueAsString(response))
        } catch (exception: Exception) {
            respond(exchange, 500, "Stub Search failed: ${exception.message}")
        }
    }

    private fun handleGet(exchange: HttpExchange) {
        try {
            if (exchange.requestMethod != "GET") return respond(exchange, 405, "Only GET")
            val issueKey = exchange.requestURI.path.substringAfterLast('/')
            issueCalls += issueKey
            if ((getFailuresRemaining[issueKey]?.getAndDecrement() ?: 0) > 0) {
                return respond(exchange, 500, "Jira GET failed: $issueKey")
            }
            val issue = issues[issueKey] ?: return respond(exchange, 404, "No Jira issue: $issueKey")
            respond(exchange, 200, objectMapper.writeValueAsString(issue))
        } catch (exception: Exception) {
            respond(exchange, 500, "Stub GET failed: ${exception.message}")
        }
    }

    private fun respond(exchange: HttpExchange, status: Int, body: String) {
        val bytes = body.toByteArray(StandardCharsets.UTF_8)
        exchange.responseHeaders.set("Content-Type", if (status == 200) "application/json" else "text/plain")
        exchange.sendResponseHeaders(status, bytes.size.toLong())
        exchange.responseBody.use { it.write(bytes) }
    }
}

private fun jiraDate(value: LocalDateTime): String =
    value.format(DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSS")) + "+0300"

private const val PAGE_SIZE = 1
private const val MAIN_KEY = "CROSSGOAL-100"
private const val SECOND_KEY = "CROSSGOAL-200"
private const val ABSENT_KEY = "CROSSGOAL-300"
private const val NEW_EPIC_KEY = "CROSSGOAL-500"
private const val QG_TASK_KEY = "CROSSGOAL-501"
private const val ANALYSIS_TASK_KEY = "CROSSGOAL-502"
private const val DEVELOPMENT_TASK_KEY = "CROSSGOAL-503"
private const val QG_CODE = "QG_ARCHITECTURE_FR2"
private const val SECOND_QG_CODE = "QG_SECURITY_FR2"
private const val UPDATED_SUMMARY = "Updated FR2 initiative"
private const val SIGMA_URL_PREFIX = "http://jira.test/browse/"

```
