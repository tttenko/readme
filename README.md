```java
@Test
fun `syncFromJira should update agent strategies from Jira issue links`() {
    // Given
    val strategy = StrategyEntity().apply {
        id = 10L
        jiraIssue = "STRAT-1"
        name = "Strategy 1"
    }

    val initiativeType = InitiativeTypeEntity().apply {
        code = "agent"
        name = "Agent"
        description = "AI Agent"
    }

    val status = StatusEntity().apply {
        code = "research"
        ordering = 1L
        name = "Research"
    }

    val agent = AIAgentEntity().apply {
        id = 1L
        agentId = "CROSSGOAL-123"
        agentJiraUrl = "CROSSGOAL-123"
        agentStatus = status
        this.initiativeType = initiativeType
    }

    val outwardIssue = GetIssueSimpleResponse(
        key = "STRAT-1",
        fields = null,
    )

    val inwardIssue = GetIssueSimpleResponse(
        key = null,
        fields = null,
    )

    val issueLink = GetIssueLinkResponse(
        id = "1",
        outwardIssue = outwardIssue,
        inwardIssue = inwardIssue,
        type = null,
    )

    val jiraResponse = IssueDto(
        id = "10001",
        key = "CROSSGOAL-123",
        fields = GetIssueFields(
            summary = "Test Agent",
            status = GetIssueStatusResponse(
                id = "1",
                name = "In Progress",
            ),
            customfield_30000 = emptyList(),
            customfield_30001 = emptyList(),
            customfield_30002 = emptyList(),
            labels = emptyList(),
            customfield_34300 = null,
            customfield_30401 = null,
            issuelinks = listOf(issueLink),
            customfield_15903 = emptyList(),
            assignee = null,
            reporter = null,
            customfield_29202 = null,
            customfield_29203 = null,
            customfield_29205 = null,
            lastViewed = null,
            resolutiondate = null,
            created = null,
            updated = null,
        ),
    )

    val agentsPage = PageImpl(listOf(agent))

    every { strategyService.findAll() } returns listOf(strategy)
    every { qualityGateRepository.findAllByDisabledIsFalse() } returns emptyList()
    every { divisionRepository.findAll() } returns emptyList()
    every { blockRepository.findAllByDisabledIsFalse() } returns emptyList()
    every { initiativeTypeRepository.findAll() } returns listOf(initiativeType)
    every { enablerRepository.findAll() } returns emptyList()

    every {
        jiraNumericValueParser.parseFirst(null)
    } returns null

    every {
        agentRepository.findAllWithPultIdAndCrossgoalPageable(any<Pageable>())
    } returns agentsPage

    every {
        jiraService.getIssue("CROSSGOAL-123", any())
    } returns jiraResponse

    every {
        jiraService.getJiraSigmaUrl()
    } returns "https://jira/"

    every {
        optionsService.findAll()
    } returns listOf(
        OptionsDto(maxResults = 100)
    )

    every {
        syncFromJiraAgentTransactionRunner.execute(eq(agent), any())
    } answers {
        val action = secondArg<(AIAgentEntity) -> Boolean>()
        action(agent)
    }

    every {
        agentRepository.save(any<AIAgentEntity>())
    } returns agent

    every {
        jiraIssueRepository.findByAgentIdAndTypeAndProject(
            any(),
            any(),
            any(),
        )
    } returns emptyList()

    every {
        jiraIssueRepository.save(any<JiraIssueEntity>())
    } answers {
        firstArg()
    }

    every {
        agentStrategyRepository.findAllByAgentId(1L)
    } returns emptyList()

    val savedStrategies = slot<List<AgentStrategyEntity>>()

    every {
        agentStrategyRepository.saveAll(capture(savedStrategies))
    } answers {
        firstArg()
    }

    every {
        agentStrategyRepository.deleteAll(any())
    } just Runs

    every {
        involvedResourceRepository.deleteAllByAiAgentId(any())
    } just Runs

    every {
        enablerRepository.deleteAllByAgentId(any())
    } just Runs

    every {
        jiraErrorTracker.reset()
    } just Runs

    // When
    syncScheduler.syncFromJira()

    // Then
    verify(exactly = 1) {
        agentStrategyRepository.saveAll(any<List<AgentStrategyEntity>>())
    }

    val savedAgentStrategies = savedStrategies.captured

    assertEquals(1, savedAgentStrategies.size)
    assertEquals(strategy, savedAgentStrategies.single().strategy)
    assertEquals(agent, savedAgentStrategies.single().agent)
    assertEquals("done", savedAgentStrategies.single().jiraLink)
}

```
