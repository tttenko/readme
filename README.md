```java

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
