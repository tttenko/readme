```java
@Test
fun `syncFromJira should process agents in batches and update them from Jira`() {
    // Given
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

    val block = BlockEntity().apply {
        code = "dev"
        shortName = "Dev"
        name = "Development"
        label = "Dev"
        disabled = false
    }

    val division = DivisionEntity().apply {
        code = "dev"
        shortName = "Dev"
        name = "Development"
        label = "Dev"
        disabled = false
        this.block = block
    }

    val agent = AIAgentEntity().apply {
        id = 1L
        agentId = "CROSSGOAL-123"
        agentJiraUrl = "CROSSGOAL-123"
        agentName = "Old Name"
        agentStatus = status
        this.initiativeType = initiativeType
    }

    val jiraResponse = IssueDto(
        id = "10001",
        key = "CROSSGOAL-123",
        fields = GetIssueFields(
            summary = "Test Agent Summary",
            status = GetIssueStatusResponse(
                id = "1",
                name = "In Progress",
            ),
            customfield_30000 = listOf("Dev"),
            customfield_30001 = listOf("Dev-Team"),
            customfield_30002 = emptyList(),
            labels = listOf("Агент"),
            customfield_34300 = "100.50",
            customfield_30401 = "200.75",
            issuelinks = emptyList(),
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

    every { strategyService.findAll() } returns emptyList()
    every { qualityGateRepository.findAllByDisabledIsFalse() } returns emptyList()
    every { divisionRepository.findAll() } returns listOf(division)
    every { blockRepository.findAllByDisabledIsFalse() } returns listOf(block)
    every { initiativeTypeRepository.findAll() } returns listOf(initiativeType)
    every { enablerRepository.findAll() } returns emptyList()

    every {
        jiraNumericValueParser.parseFirst("100.50")
    } returns BigDecimal("100.50")

    every {
        jiraNumericValueParser.parseFirst("200.75")
    } returns BigDecimal("200.75")

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
    } answers {
        firstArg()
    }

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

    every {
        agentStrategyRepository.saveAll(any<List<AgentStrategyEntity>>())
    } answers {
        firstArg()
    }

    every {
        involvedResourceRepository.deleteAllByAiAgentId(1L)
    } just Runs

    every {
        enablerRepository.deleteAllByAgentId(1L)
    } just Runs

    every {
        jiraErrorTracker.reset()
    } just Runs

    // When
    syncScheduler.syncFromJira()

    // Then
    verify(exactly = 1) {
        agentRepository.findAllWithPultIdAndCrossgoalPageable(any<Pageable>())
    }

    verify(exactly = 2) {
        agentRepository.save(agent)
    }

    assertEquals("Test Agent Summary", agent.agentName)
    assertEquals(division, agent.division)
    assertEquals(block, agent.block)
    assertEquals(initiativeType, agent.initiativeType)
    assertEquals(BigDecimal("100.50"), agent.agentEffectOptimization)
    assertEquals(BigDecimal("200.75"), agent.agentEffectRevenue)
    assertNotNull(agent.updated)
}
```
