```java

@ExtendWith(MockKExtension::class)
class JiraNewInitiativeSearchSchedulerServiceTest {

    @MockK
    private lateinit var optionsService: OptionsService

    @MockK
    private lateinit var referenceDataProvider: JiraImportReferenceDataProvider

    @MockK
    private lateinit var searchRequestFactory: JiraInitiativeSearchRequestFactory

    @MockK
    private lateinit var jiraSearchPaginator: JiraSearchPaginator

    @MockK
    private lateinit var existingJiraInitiativeRepository: ExistingJiraInitiativeRepository

    private lateinit var service: JiraNewInitiativeSearchSchedulerService

    @BeforeEach
    fun setUp() {
        service = JiraNewInitiativeSearchSchedulerService(
            optionsService = optionsService,
            referenceDataProvider = referenceDataProvider,
            searchRequestFactory = searchRequestFactory,
            jiraSearchPaginator = jiraSearchPaginator,
            existingJiraInitiativeRepository = existingJiraInitiativeRepository,
        )

        every {
            referenceDataProvider.load()
        } returns createReferenceData()

        every {
            searchRequestFactory.createNewInitiativesRequest(
                newDepth = any(),
                maxResults = any(),
                startAt = any(),
            )
        } returns mockk()
    }

    @Test
    fun `importNewInitiatives should process valid Jira initiatives in batch`() {
        val firstIssue = createJiraIssue(
            key = "CROSSGOAL-100",
        )
        val secondIssue = createJiraIssue(
            key = "CROSSGOAL-200",
        )

        val response = createSearchResponse(
            total = 2,
            issues = listOf(
                firstIssue,
                secondIssue,
            ),
        )

        every {
            optionsService.getCurrent()
        } returns OptionsDto(
            newDepth = 7,
            maxResults = 30,
        )

        every {
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf(
                    "CROSSGOAL-100",
                    "CROSSGOAL-200",
                )
            )
        } returns listOf(
            "CROSSGOAL-100"
        )

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verify(exactly = 1) {
            optionsService.getCurrent()
        }

        verify(exactly = 1) {
            referenceDataProvider.load()
        }

        verify(exactly = 1) {
            searchRequestFactory.createNewInitiativesRequest(
                newDepth = 7,
                maxResults = 30,
                startAt = 0,
            )
        }

        verify(exactly = 1) {
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf(
                    "CROSSGOAL-100",
                    "CROSSGOAL-200",
                )
            )
        }
    }

    @Test
    fun `importNewInitiatives should normalize Jira keys before existence check`() {
        val issue = createJiraIssue(
            key = "  crossgoal-100  ",
        )

        val response = createSearchResponse(
            total = 1,
            issues = listOf(issue),
        )

        every {
            optionsService.getCurrent()
        } returns OptionsDto(
            newDepth = 7,
            maxResults = 30,
        )

        every {
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf("CROSSGOAL-100")
            )
        } returns emptyList()

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verify(exactly = 1) {
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf("CROSSGOAL-100")
            )
        }
    }

    @Test
    fun `importNewInitiatives should skip cancelled initiatives`() {
        val cancelledIssue = createJiraIssue(
            key = "CROSSGOAL-100",
            statusName = "Отменена",
        )

        val response = createSearchResponse(
            total = 1,
            issues = listOf(cancelledIssue),
        )

        every {
            optionsService.getCurrent()
        } returns OptionsDto(
            newDepth = 7,
            maxResults = 30,
        )

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verify {
            existingJiraInitiativeRepository wasNot Called
        }
    }

    @Test
    fun `importNewInitiatives should skip ClassicML initiatives ignoring case`() {
        val classicMlIssue = createJiraIssue(
            key = "CROSSGOAL-100",
            labels = listOf(
                "AI_Native_портфель",
                "classicml",
            ),
        )

        val response = createSearchResponse(
            total = 1,
            issues = listOf(classicMlIssue),
        )

        every {
            optionsService.getCurrent()
        } returns OptionsDto(
            newDepth = 7,
            maxResults = 30,
        )

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verify {
            existingJiraInitiativeRepository wasNot Called
        }
    }

    @Test
    fun `importNewInitiatives should skip issues without valid CROSSGOAL key`() {
        val issueWithoutKey = createJiraIssue(
            key = null,
        )
        val issueWithBlankKey = createJiraIssue(
            key = " ",
        )
        val issueWithAnotherProjectKey = createJiraIssue(
            key = "OTHER-100",
        )

        val response = createSearchResponse(
            total = 3,
            issues = listOf(
                issueWithoutKey,
                issueWithBlankKey,
                issueWithAnotherProjectKey,
            ),
        )

        every {
            optionsService.getCurrent()
        } returns OptionsDto(
            newDepth = 7,
            maxResults = 30,
        )

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verify {
            existingJiraInitiativeRepository wasNot Called
        }
    }

    @Test
    fun `importNewInitiatives should not check existing initiatives when Jira response is empty`() {
        val response = createSearchResponse(
            total = 0,
            issues = emptyList(),
        )

        every {
            optionsService.getCurrent()
        } returns OptionsDto(
            newDepth = 7,
            maxResults = 30,
        )

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verify(exactly = 1) {
            referenceDataProvider.load()
        }

        verify(exactly = 1) {
            searchRequestFactory.createNewInitiativesRequest(
                newDepth = 7,
                maxResults = 30,
                startAt = 0,
            )
        }

        verify {
            existingJiraInitiativeRepository wasNot Called
        }
    }

    @Test
    fun `importNewInitiatives should process every Jira page separately`() {
        val firstPageIssue = createJiraIssue(
            key = "CROSSGOAL-100",
        )
        val secondPageIssue = createJiraIssue(
            key = "CROSSGOAL-200",
        )

        val firstResponse = createSearchResponse(
            total = 31,
            issues = listOf(firstPageIssue),
        )
        val secondResponse = createSearchResponse(
            total = 31,
            issues = listOf(secondPageIssue),
        )

        every {
            optionsService.getCurrent()
        } returns OptionsDto(
            newDepth = 7,
            maxResults = 30,
        )

        every {
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf("CROSSGOAL-100")
            )
        } returns emptyList()

        every {
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf("CROSSGOAL-200")
            )
        } returns emptyList()

        mockPaginator(
            maxResults = 30,
            responses = listOf(
                firstResponse,
                secondResponse,
            ),
        )

        service.importNewInitiatives()

        verify(exactly = 1) {
            referenceDataProvider.load()
        }

        verify(exactly = 1) {
            searchRequestFactory.createNewInitiativesRequest(
                newDepth = 7,
                maxResults = 30,
                startAt = 0,
            )
        }

        verify(exactly = 1) {
            searchRequestFactory.createNewInitiativesRequest(
                newDepth = 7,
                maxResults = 30,
                startAt = 30,
            )
        }

        verify(exactly = 1) {
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf("CROSSGOAL-100")
            )
        }

        verify(exactly = 1) {
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf("CROSSGOAL-200")
            )
        }
    }

    @Test
    fun `importNewInitiatives should fail when newDepth is not configured`() {
        every {
            optionsService.getCurrent()
        } returns OptionsDto(
            newDepth = null,
            maxResults = 30,
        )

        assertThrows<IllegalArgumentException> {
            service.importNewInitiatives()
        }

        verify {
            referenceDataProvider wasNot Called
            searchRequestFactory wasNot Called
            jiraSearchPaginator wasNot Called
            existingJiraInitiativeRepository wasNot Called
        }
    }

    @Test
    fun `importNewInitiatives should fail when maxResults is not configured`() {
        every {
            optionsService.getCurrent()
        } returns OptionsDto(
            newDepth = 7,
            maxResults = null,
        )

        assertThrows<IllegalArgumentException> {
            service.importNewInitiatives()
        }

        verify {
            referenceDataProvider wasNot Called
            searchRequestFactory wasNot Called
            jiraSearchPaginator wasNot Called
            existingJiraInitiativeRepository wasNot Called
        }
    }

    private fun mockPaginator(
        maxResults: Int,
        responses: List<SearchIssueResponseDto>,
    ) {
        every {
            jiraSearchPaginator.processPages(
                maxResults = maxResults,
                requestFactory = any(),
                pageProcessor = any(),
            )
        } answers {
            val requestFactory =
                secondArg<(Int) -> SearchIssueRequestDto>()

            val pageProcessor =
                thirdArg<(SearchIssueResponseDto) -> Unit>()

            responses.forEachIndexed { pageIndex, response ->
                val startAt = pageIndex * maxResults

                requestFactory(startAt)
                pageProcessor(response)
            }
        }
    }

    private fun createSearchResponse(
        total: Int,
        issues: List<SearchIssueDto>,
    ): SearchIssueResponseDto {
        return mockk<SearchIssueResponseDto> {
            every { this@mockk.total } returns total
            every { this@mockk.issues } returns issues
        }
    }

    private fun createJiraIssue(
        key: String?,
        statusName: String? = null,
        labels: List<String> = emptyList(),
    ): SearchIssueDto {
        val issue = mockk<SearchIssueDto>(
            relaxed = true
        )

        every {
            issue.key
        } returns key

        every {
            issue.fields?.status?.name
        } returns statusName

        every {
            issue.fields?.labels
        } returns labels

        return issue
    }

    private fun createReferenceData(): JiraImportReferenceData {
        return JiraImportReferenceData(
            strategies = emptyList(),
            enablersByNormalizedName = emptyMap(),
            statusesByCode = emptyMap(),
            qualityGates = emptyList(),
        )
    }
}

```
