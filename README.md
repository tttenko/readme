```java
@ExtendWith(MockKExtension::class)
class GigausageIssueUpdaterTest {

    @MockK
    lateinit var jiraIssueRepository: JiraIssueRepository

    @MockK
    lateinit var messageProvider: MessageProvider

    private lateinit var gigausageIssueUpdater: GigausageIssueUpdater

    @BeforeEach
    fun setUp() {
        gigausageIssueUpdater = GigausageIssueUpdater(
            jiraIssueRepository = jiraIssueRepository,
            messageProvider = messageProvider,
        )

        every {
            messageProvider[WRONG_WRONG_GIGAUSAGE]
        } returns "Некорректное значение gigausage"
    }

    @Test
    fun `update should delete all existing issues when raw value is null`() {
        val agent = agent(id = 1L)

        val existingIssue = jiraIssue(
            agent = agent,
            jiraKey = "GIGAUSAGE-1",
            jiraUrl = "https://jira.sberbank.ru/browse/GIGAUSAGE-1",
        )

        every {
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "gigausage",
            )
        } returns listOf(existingIssue)

        every {
            jiraIssueRepository.deleteAll(
                entities = listOf(existingIssue),
            )
        } just Runs

        gigausageIssueUpdater.update(
            agent = agent,
            rawValue = null,
        )

        verify(exactly = 1) {
            jiraIssueRepository.deleteAll(
                entities = listOf(existingIssue),
            )
        }

        verify(exactly = 0) {
            jiraIssueRepository.save(any())
        }
    }

    @Test
    fun `update should delete all existing issues when raw value is empty list`() {
        val agent = agent(id = 1L)

        val existingIssue = jiraIssue(
            agent = agent,
            jiraKey = "GIGAUSAGE-1",
            jiraUrl = "https://jira.sberbank.ru/browse/GIGAUSAGE-1",
        )

        every {
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "gigausage",
            )
        } returns listOf(existingIssue)

        every {
            jiraIssueRepository.deleteAll(
                entities = listOf(existingIssue),
            )
        } just Runs

        gigausageIssueUpdater.update(
            agent = agent,
            rawValue = emptyList<String>(),
        )

        verify(exactly = 1) {
            jiraIssueRepository.deleteAll(
                entities = listOf(existingIssue),
            )
        }

        verify(exactly = 0) {
            jiraIssueRepository.save(any())
        }
    }

    @Test
    fun `update should create new issue from jira key`() {
        val agent = agent(id = 1L)
        val savedSlot = slot<JiraIssueEntity>()

        every {
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "gigausage",
            )
        } returns emptyList()

        every {
            jiraIssueRepository.save(
                entity = capture(savedSlot),
            )
        } answers {
            savedSlot.captured
        }

        gigausageIssueUpdater.update(
            agent = agent,
            rawValue = listOf("GIGAUSAGE-123"),
        )

        val savedIssue = savedSlot.captured

        assertEquals("gigausage", savedIssue.project)
        assertEquals("initiative", savedIssue.type)
        assertEquals("GIGAUSAGE-123", savedIssue.jiraKey)
        assertEquals(
            "https://jira.sberbank.ru/browse/GIGAUSAGE-123",
            savedIssue.jiraUrl,
        )
        assertEquals(agent, savedIssue.agent)

        verify(exactly = 1) {
            jiraIssueRepository.save(
                entity = any(),
            )
        }
    }

    @Test
    fun `update should create new issue from jira url`() {
        val agent = agent(id = 1L)
        val savedSlot = slot<JiraIssueEntity>()

        val jiraUrl =
            "https://jira.sberbank.ru/browse/GIGAUSAGE-456"

        every {
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "gigausage",
            )
        } returns emptyList()

        every {
            jiraIssueRepository.save(
                entity = capture(savedSlot),
            )
        } answers {
            savedSlot.captured
        }

        gigausageIssueUpdater.update(
            agent = agent,
            rawValue = listOf(jiraUrl),
        )

        val savedIssue = savedSlot.captured

        assertEquals("gigausage", savedIssue.project)
        assertEquals("initiative", savedIssue.type)
        assertEquals("GIGAUSAGE-456", savedIssue.jiraKey)
        assertEquals(jiraUrl, savedIssue.jiraUrl)
        assertEquals(agent, savedIssue.agent)
    }

    @Test
    fun `update should update existing issue when key already exists`() {
        val agent = agent(id = 1L)

        val existingIssue = jiraIssue(
            agent = agent,
            jiraKey = "GIGAUSAGE-123",
            jiraUrl = "old-url",
        )

        every {
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "gigausage",
            )
        } returns listOf(existingIssue)

        every {
            jiraIssueRepository.save(
                entity = existingIssue,
            )
        } returns existingIssue

        gigausageIssueUpdater.update(
            agent = agent,
            rawValue = listOf("GIGAUSAGE-123"),
        )

        assertEquals("gigausage", existingIssue.project)
        assertEquals("initiative", existingIssue.type)
        assertEquals("GIGAUSAGE-123", existingIssue.jiraKey)
        assertEquals(
            "https://jira.sberbank.ru/browse/GIGAUSAGE-123",
            existingIssue.jiraUrl,
        )

        verify(exactly = 1) {
            jiraIssueRepository.save(
                entity = existingIssue,
            )
        }

        verify(exactly = 0) {
            jiraIssueRepository.delete(
                entity = any(),
            )
        }
    }

    @Test
    fun `update should delete issue missing in request`() {
        val agent = agent(id = 1L)

        val issueToKeep = jiraIssue(
            agent = agent,
            jiraKey = "GIGAUSAGE-1",
            jiraUrl = "https://jira.sberbank.ru/browse/GIGAUSAGE-1",
        )

        val issueToDelete = jiraIssue(
            agent = agent,
            jiraKey = "GIGAUSAGE-2",
            jiraUrl = "https://jira.sberbank.ru/browse/GIGAUSAGE-2",
        )

        every {
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "gigausage",
            )
        } returns listOf(
            issueToKeep,
            issueToDelete,
        )

        every {
            jiraIssueRepository.delete(
                entity = issueToDelete,
            )
        } just Runs

        every {
            jiraIssueRepository.save(
                entity = issueToKeep,
            )
        } returns issueToKeep

        gigausageIssueUpdater.update(
            agent = agent,
            rawValue = listOf("GIGAUSAGE-1"),
        )

        verify(exactly = 1) {
            jiraIssueRepository.delete(
                entity = issueToDelete,
            )
        }

        verify(exactly = 0) {
            jiraIssueRepository.delete(
                entity = issueToKeep,
            )
        }

        verify(exactly = 1) {
            jiraIssueRepository.save(
                entity = issueToKeep,
            )
        }
    }

    @Test
    fun `update should create new issue and delete missing issue`() {
        val agent = agent(id = 1L)
        val savedSlot = slot<JiraIssueEntity>()

        val issueToDelete = jiraIssue(
            agent = agent,
            jiraKey = "GIGAUSAGE-OLD",
            jiraUrl = "https://jira.sberbank.ru/browse/GIGAUSAGE-OLD",
        )

        every {
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "gigausage",
            )
        } returns listOf(issueToDelete)

        every {
            jiraIssueRepository.delete(
                entity = issueToDelete,
            )
        } just Runs

        every {
            jiraIssueRepository.save(
                entity = capture(savedSlot),
            )
        } answers {
            savedSlot.captured
        }

        gigausageIssueUpdater.update(
            agent = agent,
            rawValue = listOf("GIGAUSAGE-NEW"),
        )

        verify(exactly = 1) {
            jiraIssueRepository.delete(
                entity = issueToDelete,
            )
        }

        val savedIssue = savedSlot.captured

        assertEquals("GIGAUSAGE-NEW", savedIssue.jiraKey)
        assertEquals(
            "https://jira.sberbank.ru/browse/GIGAUSAGE-NEW",
            savedIssue.jiraUrl,
        )
    }

    @Test
    fun `update should trim values before saving`() {
        val agent = agent(id = 1L)
        val savedSlot = slot<JiraIssueEntity>()

        every {
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "gigausage",
            )
        } returns emptyList()

        every {
            jiraIssueRepository.save(
                entity = capture(savedSlot),
            )
        } answers {
            savedSlot.captured
        }

        gigausageIssueUpdater.update(
            agent = agent,
            rawValue = listOf("  GIGAUSAGE-777  "),
        )

        assertEquals("GIGAUSAGE-777", savedSlot.captured.jiraKey)
        assertEquals(
            "https://jira.sberbank.ru/browse/GIGAUSAGE-777",
            savedSlot.captured.jiraUrl,
        )
    }

    @Test
    fun `update should process duplicate keys only once`() {
        val agent = agent(id = 1L)

        every {
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = 1L,
                type = "initiative",
                project = "gigausage",
            )
        } returns emptyList()

        every {
            jiraIssueRepository.save(
                entity = any(),
            )
        } answers {
            firstArg()
        }

        gigausageIssueUpdater.update(
            agent = agent,
            rawValue = listOf(
                "GIGAUSAGE-1",
                "https://jira.sberbank.ru/browse/GIGAUSAGE-1",
            ),
        )

        /*
         * associateBy оставит одно значение на один jiraKey.
         */
        verify(exactly = 1) {
            jiraIssueRepository.save(
                entity = any(),
            )
        }
    }

    @Test
    fun `update should throw exception when raw value is not list`() {
        val agent = agent(id = 1L)

        val exception = assertThrows<AiBadRequestException> {
            gigausageIssueUpdater.update(
                agent = agent,
                rawValue = "GIGAUSAGE-1",
            )
        }

        assertEquals(WRONG_WRONG_GIGAUSAGE, exception.errorCode)
        assertEquals("Некорректное значение gigausage", exception.message)

        verify(exactly = 0) {
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = any(),
                type = any(),
                project = any(),
            )
        }
    }

    @Test
    fun `update should throw exception when list item is not string`() {
        val agent = agent(id = 1L)

        val exception = assertThrows<AiBadRequestException> {
            gigausageIssueUpdater.update(
                agent = agent,
                rawValue = listOf(123),
            )
        }

        assertEquals(WRONG_WRONG_GIGAUSAGE, exception.errorCode)
    }

    @Test
    fun `update should throw exception when value is blank string`() {
        val agent = agent(id = 1L)

        val exception = assertThrows<AiBadRequestException> {
            gigausageIssueUpdater.update(
                agent = agent,
                rawValue = listOf("   "),
            )
        }

        assertEquals(WRONG_WRONG_GIGAUSAGE, exception.errorCode)
    }

    @Test
    fun `update should throw exception when value has wrong prefix`() {
        val agent = agent(id = 1L)

        val exception = assertThrows<AiBadRequestException> {
            gigausageIssueUpdater.update(
                agent = agent,
                rawValue = listOf("ABC-123"),
            )
        }

        assertEquals(WRONG_WRONG_GIGAUSAGE, exception.errorCode)
    }

    @Test
    fun `update should throw exception when jira url does not contain gigausage key`() {
        val agent = agent(id = 1L)

        val exception = assertThrows<AiBadRequestException> {
            gigausageIssueUpdater.update(
                agent = agent,
                rawValue = listOf(
                    "https://jira.sberbank.ru/browse/ABC-123"
                ),
            )
        }

        assertEquals(WRONG_WRONG_GIGAUSAGE, exception.errorCode)
    }

    private fun agent(
        id: Long,
    ): AIAgentEntity {
        return mockk<AIAgentEntity>(relaxed = true) {
            every {
                this@mockk.id
            } returns id
        }
    }

    private fun jiraIssue(
        agent: AIAgentEntity,
        jiraKey: String,
        jiraUrl: String,
    ): JiraIssueEntity {
        return JiraIssueEntity(
            project = "gigausage",
            type = "initiative",
            agent = agent,
        ).apply {
            this.jiraKey = jiraKey
            this.jiraUrl = jiraUrl
        }
    }
}

  
```
