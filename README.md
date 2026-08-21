```java
@Test
fun `syncFromJira should update jira issues for tasks`() {
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
        name = "Test Task 1"
        disabled = false
        type = QualityGateType.quality_gate
    }

    val monitoringLink = GetIssueLinkResponse(
        id = "1",
        outwardIssue = GetIssueSimpleResponse(
            id = "30001",
            key = "MONITORING-789",
            fields = GetIssueFieldsSimple(
                summary = "AI мониторинг Epic",
                status = null,
                priority = null,
                issuetype = null,
            ),
        ),
        inwardIssue = null,
        type = null,
    )

    val monitoringEpic = JiraIssueEntity(
        agent = agent,
        type = "epic",
        project = "crossgoal",
        jiraKey = "MONITORING-789",
        jiraId = "30001",
    ).apply {
        id = 2L
    }

    val jiraResponse = IssueDto(
        id = "10001",
        key = "CROSSGOAL-123",
        fields = GetIssueFields(
            summary = "Test Task 1",
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

    val task = SearchIssueDto(
        id = "20001",
        key = "TASK-1",
        fields = SearchIssueFieldsDto(
            summary = "Test Task 1",
            status = SearchIssueStatusDto(
                id = "1",
                name = "In Progress",
            ),
            labels = emptyList(),
            customfield_15903 = emptyList(),
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
    } returns listOf(task)

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

    // jira_issue самой инициативы
    every {
        jiraIssueRepository.findByAgentIdAndTypeAndProject(
            1L,
            JiraIssueType.initiative.name,
            "crossgoal",
        )
    } returns emptyList()

    // monitoring epic
    every {
        jiraIssueRepository.findByAgentIdAndTypeAndProject(
            1L,
            JiraIssueType.epic.name,
            "crossgoal",
        )
    } returns listOf(monitoringEpic)

    every {
        jiraIssueRepository.save(any<JiraIssueEntity>())
    } answers {
        firstArg()
    }

    // TASK-1 ещё не существует в БД
    every {
        jiraIssueRepository.findAllByAgentIdAndJiraKeyIn(
            1L,
            setOf("TASK-1"),
        )
    } returns emptyList()

    val savedJiraIssues = slot<Iterable<JiraIssueEntity>>()

    every {
        jiraIssueRepository.saveAll(capture(savedJiraIssues))
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

    every {
        aiagentQualityGateService.updateState(any(), any())
    } just Runs

    every {
        jiraErrorTracker.reset()
    } just Runs

    // When
    syncScheduler.syncFromJira()

    // Then
    verify(exactly = 1) {
        jiraIssueRepository.saveAll(any<Iterable<JiraIssueEntity>>())
    }

    val savedTaskIssues = savedJiraIssues.captured.toList()

    assertEquals(1, savedTaskIssues.size)

    with(savedTaskIssues.single()) {
        assertEquals("TASK-1", jiraKey)
        assertEquals("20001", jiraId)
        assertEquals("task", type)
        assertEquals("crossgoal", project)
        assertEquals(2L, parentId)
        assertEquals(qualityGate, this.qualityGate)
        assertEquals("https://jira/TASK-1", jiraUrl)
    }

```
