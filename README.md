```java
@ExtendWith(MockitoExtension::class)
class JiraNewInitiativeSearchSchedulerServiceTest {

    @Mock
    private lateinit var optionsService: OptionsService

    @Mock
    private lateinit var referenceDataProvider: JiraImportReferenceDataProvider

    @Mock
    private lateinit var searchRequestFactory: JiraInitiativeSearchRequestFactory

    @Mock
    private lateinit var jiraSearchPaginator: JiraSearchPaginator

    @Mock
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

        whenever(referenceDataProvider.load())
            .thenReturn(createReferenceData())

        whenever(
            searchRequestFactory.createNewInitiativesRequest(
                any(),
                any(),
                any(),
            )
        ).thenReturn(mock())
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

        whenever(optionsService.getCurrent())
            .thenReturn(
                OptionsDto(
                    newDepth = 7,
                    maxResults = 30,
                )
            )

        whenever(
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf(
                    "CROSSGOAL-100",
                    "CROSSGOAL-200",
                )
            )
        ).thenReturn(
            listOf("CROSSGOAL-100")
        )

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verify(optionsService).getCurrent()
        verify(referenceDataProvider).load()

        verify(searchRequestFactory)
            .createNewInitiativesRequest(
                newDepth = 7,
                maxResults = 30,
                startAt = 0,
            )

        verify(existingJiraInitiativeRepository)
            .findExistingJiraKeys(
                setOf(
                    "CROSSGOAL-100",
                    "CROSSGOAL-200",
                )
            )
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

        whenever(optionsService.getCurrent())
            .thenReturn(
                OptionsDto(
                    newDepth = 7,
                    maxResults = 30,
                )
            )

        whenever(
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf("CROSSGOAL-100")
            )
        ).thenReturn(emptyList())

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verify(existingJiraInitiativeRepository)
            .findExistingJiraKeys(
                setOf("CROSSGOAL-100")
            )
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

        whenever(optionsService.getCurrent())
            .thenReturn(
                OptionsDto(
                    newDepth = 7,
                    maxResults = 30,
                )
            )

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verifyNoInteractions(existingJiraInitiativeRepository)
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

        whenever(optionsService.getCurrent())
            .thenReturn(
                OptionsDto(
                    newDepth = 7,
                    maxResults = 30,
                )
            )

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verifyNoInteractions(existingJiraInitiativeRepository)
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

        whenever(optionsService.getCurrent())
            .thenReturn(
                OptionsDto(
                    newDepth = 7,
                    maxResults = 30,
                )
            )

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verifyNoInteractions(existingJiraInitiativeRepository)
    }

    @Test
    fun `importNewInitiatives should not check existing initiatives when Jira response is empty`() {
        val response = createSearchResponse(
            total = 0,
            issues = emptyList(),
        )

        whenever(optionsService.getCurrent())
            .thenReturn(
                OptionsDto(
                    newDepth = 7,
                    maxResults = 30,
                )
            )

        mockPaginator(
            maxResults = 30,
            responses = listOf(response),
        )

        service.importNewInitiatives()

        verify(referenceDataProvider).load()

        verify(searchRequestFactory)
            .createNewInitiativesRequest(
                newDepth = 7,
                maxResults = 30,
                startAt = 0,
            )

        verifyNoInteractions(existingJiraInitiativeRepository)
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

        whenever(optionsService.getCurrent())
            .thenReturn(
                OptionsDto(
                    newDepth = 7,
                    maxResults = 30,
                )
            )

        whenever(
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf("CROSSGOAL-100")
            )
        ).thenReturn(emptyList())

        whenever(
            existingJiraInitiativeRepository.findExistingJiraKeys(
                setOf("CROSSGOAL-200")
            )
        ).thenReturn(emptyList())

        mockPaginator(
            maxResults = 30,
            responses = listOf(
                firstResponse,
                secondResponse,
            ),
        )

        service.importNewInitiatives()

        verify(referenceDataProvider, times(1))
            .load()

        verify(searchRequestFactory)
            .createNewInitiativesRequest(
                newDepth = 7,
                maxResults = 30,
                startAt = 0,
            )

        verify(searchRequestFactory)
            .createNewInitiativesRequest(
                newDepth = 7,
                maxResults = 30,
                startAt = 30,
            )

        verify(existingJiraInitiativeRepository)
            .findExistingJiraKeys(
                setOf("CROSSGOAL-100")
            )

        verify(existingJiraInitiativeRepository)
            .findExistingJiraKeys(
                setOf("CROSSGOAL-200")
            )
    }

    @Test
    fun `importNewInitiatives should fail when newDepth is not configured`() {
        whenever(optionsService.getCurrent())
            .thenReturn(
                OptionsDto(
                    newDepth = null,
                    maxResults = 30,
                )
            )

        assertThrows<IllegalArgumentException> {
            service.importNewInitiatives()
        }

        verifyNoInteractions(
            referenceDataProvider,
            searchRequestFactory,
            jiraSearchPaginator,
            existingJiraInitiativeRepository,
        )
    }

    @Test
    fun `importNewInitiatives should fail when maxResults is not configured`() {
        whenever(optionsService.getCurrent())
            .thenReturn(
                OptionsDto(
                    newDepth = 7,
                    maxResults = null,
                )
            )

        assertThrows<IllegalArgumentException> {
            service.importNewInitiatives()
        }

        verifyNoInteractions(
            referenceDataProvider,
            searchRequestFactory,
            jiraSearchPaginator,
            existingJiraInitiativeRepository,
        )
    }

    private fun mockPaginator(
        maxResults: Int,
        responses: List<SearchIssueResponseDto>,
    ) {
        doAnswer { invocation ->
            val requestFactory =
                invocation.getArgument<(Int) -> SearchIssueRequestDto>(1)

            val pageProcessor =
                invocation.getArgument<(SearchIssueResponseDto) -> Unit>(2)

            responses.forEachIndexed { pageIndex, response ->
                val startAt = pageIndex * maxResults

                requestFactory(startAt)
                pageProcessor(response)
            }

            null
        }.whenever(jiraSearchPaginator)
            .processPages(
                eq(maxResults),
                any(),
                any(),
            )
    }

    private fun createSearchResponse(
        total: Int,
        issues: List<SearchIssueDto>,
    ): SearchIssueResponseDto {
        return mock<SearchIssueResponseDto>().also { response ->
            whenever(response.total)
                .thenReturn(total)

            whenever(response.issues)
                .thenReturn(issues)
        }
    }

    private fun createJiraIssue(
        key: String?,
        statusName: String? = null,
        labels: List<String> = emptyList(),
    ): SearchIssueDto {
        val issue = Mockito.mock(
            SearchIssueDto::class.java,
            Answers.RETURNS_DEEP_STUBS,
        )

        whenever(issue.key)
            .thenReturn(key)

        whenever(issue.fields?.status?.name)
            .thenReturn(statusName)

        whenever(issue.fields?.labels)
            .thenReturn(labels)

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
