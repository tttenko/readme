```java
@Test
fun `should log error when searchTasksByEpicKey throw`() {
    // Given
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
    }

    val qualityGate = QualityGateEntity().apply {
        code = "QG1"
        name = "QG1"
        disabled = false
        type = QualityGateType.quality_gate
    }

    val monitoringLink = GetIssueLinkResponse(
        id = "1",
        outwardIssue = null,
        inwardIssue = GetIssueSimpleResponse(
            id = "30001",
            key = "MONITORING-789",
            fields = GetIssueFieldsSimple(
                summary = "AI мониторинг Epic",
                status = null,
                priority = null,
                issuetype = null,
            ),
        ),
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
            labels = listOf("AI-не-эффективность"),
            customfield_34300 = null,
            customfield_30401 = null,
            customfield_31304 = null,
            customfield_31305 = null,
            customfield_31306 = null,
            customfield_31307 = null,
            issuelinks = listOf(monitoringLink),
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
    every { statusRepository.findFirstByCode(any()) } returns status
    every { qualityGateRepository.findAllByDisabledIsFalse() } returns listOf(qualityGate)
    every { divisionRepository.findAll() } returns emptyList()
    every { blockRepository.findAllByDisabledIsFalse() } returns emptyList()
    every { initiativeTypeRepository.findAll() } returns emptyList()
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
        jiraService.searchTasksByEpicKey("MONITORING-789", 100)
    } throws RuntimeException("Jira search error")

    every {
        jiraService.getJiraSigmaUrl()
    } returns "https://jira/"

    every {
        optionsService.findAll()
    } returns listOf(
        OptionsDto(maxResults = 100)
    )

    every {
        jiraErrorTracker.increment()
    } returns 1

    every {
        jiraErrorTracker.reset()
    } just Runs

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
        agentStrategyRepository.saveAll(any<List<AgentStrategyEntity>>())
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
        agentStatusSlaRepository.save(any())
    } answers {
        firstArg()
    }

    // When
    syncScheduler.syncFromJira()

    // Then
    verify(exactly = 1) {
        jiraService.searchTasksByEpicKey(
            "MONITORING-789",
            100,
        )
    }

    verify(exactly = 1) {
        jiraErrorTracker.increment()
    }

    // Ошибка загрузки tasks не прерывает обработку:
    // сохраняется jira_issue инициативы и monitoring epic.
    verify(exactly = 2) {
        jiraIssueRepository.save(any<JiraIssueEntity>())
    }
}

```
