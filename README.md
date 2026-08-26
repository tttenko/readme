```java
import io.mockk.InjectMockKs
import io.mockk.MockK
import io.mockk.every
import io.mockk.just
import io.mockk.Runs
import io.mockk.mockk
import io.mockk.verify
import io.mockk.junit5.MockKExtension
import org.junit.jupiter.api.Assertions.assertFalse
import org.junit.jupiter.api.Assertions.assertSame
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.ExtendWith

@ExtendWith(MockKExtension::class)
class JiraNewInitiativeMonitoringServiceTest {

    @MockK
    private lateinit var monitoringEpicResolver: JiraMonitoringEpicResolver

    @MockK
    private lateinit var monitoringTaskSearchService: JiraMonitoringTaskSearchService

    @MockK
    private lateinit var taskQualityGateMatcher: JiraTaskQualityGateMatcher

    @MockK
    private lateinit var monitoringPersistenceService: JiraMonitoringPersistenceService

    @InjectMockKs
    private lateinit var service: JiraNewInitiativeMonitoringService

    private val referenceData = JiraImportReferenceData(
        strategiesByJiraKey = emptyMap(),
        enablersByNormalizedName = emptyMap(),
        statusesByCode = emptyMap(),
        qualityGates = emptyList(),
        divisionsByLabel = emptyMap(),
        blocksByLabel = emptyMap(),
        initiativeTypesByCode = emptyMap(),
    )

    @Test
    fun `should finish synchronization without monitoring for AI effectiveness initiative`() {
        val issue = jiraIssue(
            jiraKey = "CROSSGOAL-100",
            labels = listOf("AI-эффективность"),
        )

        every {
            monitoringEpicResolver.isMonitoringRequired(
                labels = listOf("AI-эффективность"),
            )
        } returns false

        every {
            monitoringPersistenceService.markSynchronizationDone(
                agentId = 100L,
                jiraKey = "CROSSGOAL-100",
            )
        } just Runs

        val result = service.synchronizeMonitoring(
            agentId = 100L,
            issue = issue,
            referenceData = referenceData,
            maxResults = 30,
            jiraErrorTracker = JiraErrorTracker(),
        )

        assertTrue(result)

        verify(exactly = 1) {
            monitoringPersistenceService.markSynchronizationDone(
                agentId = 100L,
                jiraKey = "CROSSGOAL-100",
            )
        }

        verify(exactly = 0) {
            monitoringEpicResolver.findMonitoringEpic(any(), any())
        }

        verify(exactly = 0) {
            monitoringTaskSearchService.searchMonitoringTasks(
                epicKey = any(),
                maxResults = any(),
                jiraErrorTracker = any(),
            )
        }
    }

    @Test
    fun `should finish synchronization when monitoring epic is absent`() {
        val issue = jiraIssue(
            jiraKey = "CROSSGOAL-101",
        )

        every {
            monitoringEpicResolver.isMonitoringRequired(any())
        } returns true

        every {
            monitoringEpicResolver.findMonitoringEpic(
                initiativeJiraKey = "CROSSGOAL-101",
                issueLinks = any(),
            )
        } returns null

        every {
            monitoringPersistenceService.markSynchronizationDone(
                agentId = 101L,
                jiraKey = "CROSSGOAL-101",
            )
        } just Runs

        val result = service.synchronizeMonitoring(
            agentId = 101L,
            issue = issue,
            referenceData = referenceData,
            maxResults = 30,
            jiraErrorTracker = JiraErrorTracker(),
        )

        assertTrue(result)

        verify(exactly = 1) {
            monitoringPersistenceService.markSynchronizationDone(
                agentId = 101L,
                jiraKey = "CROSSGOAL-101",
            )
        }

        verify(exactly = 0) {
            monitoringTaskSearchService.searchMonitoringTasks(
                epicKey = any(),
                maxResults = any(),
                jiraErrorTracker = any(),
            )
        }
    }

    @Test
    fun `should mark synchronization as error when monitoring task search fails`() {
        val issue = jiraIssue(
            jiraKey = "CROSSGOAL-102",
        )

        val monitoringEpic = JiraMonitoringEpicData(
            jiraId = "500",
            jiraKey = "CROSSGOAL-500",
        )

        every {
            monitoringEpicResolver.isMonitoringRequired(any())
        } returns true

        every {
            monitoringEpicResolver.findMonitoringEpic(
                initiativeJiraKey = "CROSSGOAL-102",
                issueLinks = any(),
            )
        } returns monitoringEpic

        every {
            monitoringPersistenceService.saveMonitoringEpic(
                agentId = 102L,
                monitoringEpic = monitoringEpic,
            )
        } returns 500L

        every {
            monitoringTaskSearchService.searchMonitoringTasks(
                epicKey = "CROSSGOAL-500",
                maxResults = 30,
                jiraErrorTracker = any(),
            )
        } throws RuntimeException("Jira unavailable")

        every {
            monitoringPersistenceService.markSynchronizationError(
                agentId = 102L,
                jiraKey = "CROSSGOAL-102",
            )
        } just Runs

        val result = service.synchronizeMonitoring(
            agentId = 102L,
            issue = issue,
            referenceData = referenceData,
            maxResults = 30,
            jiraErrorTracker = JiraErrorTracker(),
        )

        assertFalse(result)

        verify(exactly = 1) {
            monitoringPersistenceService.saveMonitoringEpic(
                agentId = 102L,
                monitoringEpic = monitoringEpic,
            )
        }

        verify(exactly = 1) {
            monitoringPersistenceService.markSynchronizationError(
                agentId = 102L,
                jiraKey = "CROSSGOAL-102",
            )
        }

        verify(exactly = 0) {
            monitoringPersistenceService.saveMonitoringData(
                agentId = any(),
                monitoringEpicIssueId = any(),
                monitoringEpicKey = any(),
                initiativeJiraKey = any(),
                taskMatches = any(),
                referenceData = any(),
            )
        }
    }

    @Test
    fun `should mark synchronization as error and rethrow when Jira error limit is exceeded`() {
        val issue = jiraIssue(
            jiraKey = "CROSSGOAL-103",
        )

        val monitoringEpic = JiraMonitoringEpicData(
            jiraId = "501",
            jiraKey = "CROSSGOAL-501",
        )

        val expectedException =
            JiraErrorLimitExceededException(
                RuntimeException("Jira unavailable")
            )

        every {
            monitoringEpicResolver.isMonitoringRequired(any())
        } returns true

        every {
            monitoringEpicResolver.findMonitoringEpic(
                initiativeJiraKey = "CROSSGOAL-103",
                issueLinks = any(),
            )
        } returns monitoringEpic

        every {
            monitoringPersistenceService.saveMonitoringEpic(
                agentId = 103L,
                monitoringEpic = monitoringEpic,
            )
        } returns 501L

        every {
            monitoringTaskSearchService.searchMonitoringTasks(
                epicKey = "CROSSGOAL-501",
                maxResults = 30,
                jiraErrorTracker = any(),
            )
        } throws expectedException

        every {
            monitoringPersistenceService.markSynchronizationError(
                agentId = 103L,
                jiraKey = "CROSSGOAL-103",
            )
        } just Runs

        val actualException =
            assertThrows<JiraErrorLimitExceededException> {
                service.synchronizeMonitoring(
                    agentId = 103L,
                    issue = issue,
                    referenceData = referenceData,
                    maxResults = 30,
                    jiraErrorTracker = JiraErrorTracker(),
                )
            }

        assertSame(expectedException, actualException)

        verify(exactly = 1) {
            monitoringPersistenceService.markSynchronizationError(
                agentId = 103L,
                jiraKey = "CROSSGOAL-103",
            )
        }
    }

    private fun jiraIssue(
        jiraKey: String,
        labels: List<String> = emptyList(),
    ): SearchIssueDto {
        val fields = mockk<SearchIssueFieldsDto> {
            every { this@mockk.labels } returns labels
            every { issuelinks } returns emptyList()
        }

        return mockk {
            every { key } returns jiraKey
            every { this@mockk.fields } returns fields
        }
    }
}


@ExtendWith(MockKExtension::class)
class JiraNewInitiativeImportServiceTest {

    @MockK
    private lateinit var optionsService: OptionsService

    @MockK
    private lateinit var jiraIssueKeyExtractor: JiraIssueKeyExtractor

    @MockK
    private lateinit var referenceDataProvider: JiraImportReferenceDataProvider

    @MockK
    private lateinit var searchRequestFactory: JiraInitiativeSearchRequestFactory

    @MockK
    private lateinit var jiraSearchPaginator: JiraSearchPaginator

    @MockK
    private lateinit var existingJiraInitiativeRepository: ExistingJiraInitiativeRepository

    @MockK
    private lateinit var jiraNewInitiativeCreator: JiraNewInitiativeCreationService

    @MockK
    private lateinit var jiraNewInitiativeMonitoringService: JiraNewInitiativeMonitoringService

    @InjectMockKs
    private lateinit var service: JiraNewInitiativeImportService

    private val referenceData = JiraImportReferenceData(
        strategiesByJiraKey = emptyMap(),
        enablersByNormalizedName = emptyMap(),
        statusesByCode = emptyMap(),
        qualityGates = emptyList(),
        divisionsByLabel = emptyMap(),
        blocksByLabel = emptyMap(),
        initiativeTypesByCode = emptyMap(),
    )

    @BeforeEach
    fun setUp() {
        val options = mockk<OptionsDto> {
            every { newDepth } returns 7
            every { maxResults } returns 30
        }

        every {
            optionsService.getCurrent()
        } returns options

        every {
            referenceDataProvider.load()
        } returns referenceData
    }

    @Test
    fun `should skip cancelled Jira initiative`() {
        val issue = jiraIssue(
            jiraKey = "CROSSGOAL-200",
            statusName = "Отменена",
        )

        every {
            jiraIssueKeyExtractor.extractCrossgoalKey(
                "CROSSGOAL-200"
            )
        } returns "CROSSGOAL-200"

        returnSinglePage(issue)

        service.importNewInitiatives()

        verify(exactly = 0) {
            existingJiraInitiativeRepository.findExistingJiraKeys(any())
        }

        verify(exactly = 0) {
            jiraNewInitiativeCreator.createInitiativeFromJira(
                any(),
                any(),
            )
        }

        verify(exactly = 0) {
            jiraNewInitiativeMonitoringService.synchronizeMonitoring(
                agentId = any(),
                issue = any(),
                referenceData = any(),
                maxResults = any(),
                jiraErrorTracker = any(),
            )
        }
    }

    @Test
    fun `should skip ClassicML Jira initiative`() {
        val issue = jiraIssue(
            jiraKey = "CROSSGOAL-201",
            statusName = "В работе",
            labels = listOf(
                "AI_Native_портфель",
                "ClassicML",
            ),
        )

        every {
            jiraIssueKeyExtractor.extractCrossgoalKey(
                "CROSSGOAL-201"
            )
        } returns "CROSSGOAL-201"

        returnSinglePage(issue)

        service.importNewInitiatives()

        verify(exactly = 0) {
            existingJiraInitiativeRepository.findExistingJiraKeys(any())
        }

        verify(exactly = 0) {
            jiraNewInitiativeCreator.createInitiativeFromJira(
                any(),
                any(),
            )
        }

        verify(exactly = 0) {
            jiraNewInitiativeMonitoringService.synchronizeMonitoring(
                agentId = any(),
                issue = any(),
                referenceData = any(),
                maxResults = any(),
                jiraErrorTracker = any(),
            )
        }
    }

    @Test
    fun `should not start monitoring when initiative creation was skipped`() {
        val issue = jiraIssue(
            jiraKey = "CROSSGOAL-202",
            statusName = "В работе",
            labels = listOf("AI_Native_портфель"),
        )

        every {
            jiraIssueKeyExtractor.extractCrossgoalKey(
                "CROSSGOAL-202"
            )
        } returns "CROSSGOAL-202"

        every {
            existingJiraInitiativeRepository.findExistingJiraKeys(
                listOf("CROSSGOAL-202")
            )
        } returns emptyList()

        every {
            jiraNewInitiativeCreator.createInitiativeFromJira(
                issue = issue,
                referenceData = referenceData,
            )
        } returns null

        returnSinglePage(issue)

        service.importNewInitiatives()

        verify(exactly = 1) {
            jiraNewInitiativeCreator.createInitiativeFromJira(
                issue = issue,
                referenceData = referenceData,
            )
        }

        verify(exactly = 0) {
            jiraNewInitiativeMonitoringService.synchronizeMonitoring(
                agentId = any(),
                issue = any(),
                referenceData = any(),
                maxResults = any(),
                jiraErrorTracker = any(),
            )
        }
    }

@Test
fun `should skip initiative when block and division cannot be resolved`() {
    val fields = mockk<SearchIssueFieldsDto> {
        every { customfield_30000 } returns emptyList()
        every { customfield_30001 } returns emptyList()
    }

    val issue = mockk<SearchIssueDto> {
        every { key } returns "CROSSGOAL-300"
        every { this@mockk.fields } returns fields
    }

    val referenceData = JiraImportReferenceData(
        strategiesByJiraKey = emptyMap(),
        enablersByNormalizedName = emptyMap(),
        statusesByCode = emptyMap(),
        qualityGates = emptyList(),
        divisionsByLabel = emptyMap(),
        blocksByLabel = emptyMap(),
        initiativeTypesByCode = emptyMap(),
    )

    every {
        organizationResolver.resolveOrganization(
            initiatorUnits = emptyList(),
            executorUnits = emptyList(),
            referenceData = referenceData,
        )
    } returns JiraInitiativeOrganization(
        division = null,
        block = null,
    )

    val result =
        service.createInitiativeFromJira(
            issue = issue,
            referenceData = referenceData,
        )

    assertNull(result)

    verify(exactly = 0) {
        agentRepository.save(any())
    }
}

    private fun returnSinglePage(
        issue: SearchIssueDto,
    ) {
        every {
            jiraSearchPaginator.processPages(
                maxResults = any(),
                jiraErrorTracker = any(),
                requestFactory = any(),
                pageProcessor = any(),
            )
        } answers {
            val pageProcessor =
                arg<(SearchIssueResponseDto) -> Unit>(3)

            pageProcessor(
                SearchIssueResponseDto(
                    startAt = 0,
                    maxResults = 30,
                    total = 1,
                    issues = listOf(issue),
                )
            )
        }
    }

    private fun jiraIssue(
        jiraKey: String,
        statusName: String?,
        labels: List<String> = emptyList(),
    ): SearchIssueDto {

        val status =
            statusName?.let { value ->
                mockk<SearchIssueStatusDto> {
                    every { name } returns value
                }
            }

        val fields =
            mockk<SearchIssueFieldsDto> {
                every { this@mockk.status } returns status
                every { this@mockk.labels } returns labels
            }

        return mockk {
            every { key } returns jiraKey
            every { this@mockk.fields } returns fields
        }
    }
}


```
