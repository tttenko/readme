```java
@Test
fun `syncFromJira should handle involved resources with null and numeric values`() {
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
            customfield_31304 = "10",
            customfield_31305 = null,
            customfield_31306 = "5",
            customfield_31307 = " ",
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
    every { divisionRepository.findAll() } returns emptyList()
    every { blockRepository.findAllByDisabledIsFalse() } returns emptyList()
    every { initiativeTypeRepository.findAll() } returns emptyList()
    every { enablerRepository.findAll() } returns emptyList()

    every {
        jiraNumericValueParser.parseFirst("10")
    } returns BigDecimal("10")

    every {
        jiraNumericValueParser.parseFirst("5")
    } returns BigDecimal("5")

    every {
        jiraNumericValueParser.parseFirst(null)
    } returns null

    every {
        jiraNumericValueParser.parseFirst(" ")
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
        secondArg<(AIAgentEntity) -> Boolean>()(agent)
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
        agentStrategyRepository.findAllByAgentId(any())
    } returns emptyList()

    every {
        agentStrategyRepository.saveAll(any<List<AgentStrategyEntity>>())
    } answers {
        firstArg()
    }

    every {
        involvedResourceRepository.deleteAllByAiAgentId(1L)
    } just Runs

    val savedResources = slot<List<InvolvedResourceEntity>>()

    every {
        involvedResourceRepository.saveAll(capture(savedResources))
    } answers {
        firstArg()
    }

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
        involvedResourceRepository.deleteAllByAiAgentId(1L)
    }

    verify(exactly = 1) {
        involvedResourceRepository.saveAll(any<List<InvolvedResourceEntity>>())
    }

    val resources = savedResources.captured

    assertEquals(2, resources.size)

    val withoutSteerCoBusiness = resources.single {
        it.id.source == "without_steerCo" &&
            it.id.type == "business"
    }

    assertEquals(1L, withoutSteerCoBusiness.id.aiAgentId)
    assertEquals(BigDecimal("10"), withoutSteerCoBusiness.value)

    val steerCoBusiness = resources.single {
        it.id.source == "steerCo" &&
            it.id.type == "business"
    }

    assertEquals(1L, steerCoBusiness.id.aiAgentId)
    assertEquals(BigDecimal("5"), steerCoBusiness.value)
}
```
