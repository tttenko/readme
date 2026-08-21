```java
@Test
fun `syncFromJira should update enablers from Jira`() {
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

    val agent = AIAgentEntity().apply {
        id = 1L
        agentId = "CROSSGOAL-123"
        agentJiraUrl = "CROSSGOAL-123"
        agentStatus = status
        this.initiativeType = initiativeType
    }

    val enabler1 = GetCheckboxOptionItem(
        name = "GIGACHAT",
        checked = true,
    )

    val enabler2 = GetCheckboxOptionItem(
        name = "DATA_LENS",
        checked = false,
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
            issuelinks = emptyList(),
            customfield_15903 = listOf(enabler1, enabler2),
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

    val enablerEntity1 = EnablerEntity().apply {
        id = 10L
        name = "GIGACHAT"
        disabled = false
    }

    val enablerEntity2 = EnablerEntity().apply {
        id = 20L
        name = "DATA_LENS"
        disabled = false
    }

    val agentsPage = PageImpl(listOf(agent))

    every { strategyService.findAll() } returns emptyList()
    every { qualityGateRepository.findAllByDisabledIsFalse() } returns emptyList()
    every { divisionRepository.findAll() } returns emptyList()
    every { blockRepository.findAllByDisabledIsFalse() } returns emptyList()
    every { initiativeTypeRepository.findAll() } returns listOf(initiativeType)

    every {
        enablerRepository.findAll()
    } returns listOf(enablerEntity1, enablerEntity2)

    every {
        enablerNameNormalizer.normalize("GIGACHAT")
    } returns "gigachat"

    every {
        enablerNameNormalizer.normalize("DATA_LENS")
    } returns "data_lens"

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
        agentStrategyRepository.findAllByAgentId(any())
    } returns emptyList()

    every {
        agentStrategyRepository.saveAll(
            any<List<AgentStrategyEntity>>()
        )
    } answers {
        firstArg()
    }

    every {
        involvedResourceRepository.deleteAllByAiAgentId(any())
    } just Runs

    every {
        enablerRepository.deleteAllByAgentId(any())
    } just Runs

    every {
        enablerRepository.addAllToAgent(any(), any())
    } just Runs

    every {
        jiraErrorTracker.reset()
    } just Runs

    // When
    syncScheduler.syncFromJira()

    // Then
    verify(exactly = 1) {
        enablerRepository.deleteAllByAgentId(1L)
    }

    verify(exactly = 1) {
        enablerRepository.addAllToAgent(
            1L,
            listOf(10L),
        )
    }
}

```
