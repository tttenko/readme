```java

@Test
fun `getAIAgentList should return hasMetricsValue true when status is targetSolution and agent has supported metric type`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "targetSolution",
    )

    every {
        userInfoProvider.currentUser()
    } returns cmsAdminUser()

    mockAgentRepository(
        agents = listOf(agent),
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes,
        )
    } returns setOf(1L)

    val result = service.getAIAgentList(request)

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].hasMetricsValue).isTrue()

    verify(exactly = 1) {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes,
        )
    }
}

@Test
fun `getAIAgentList should return hasMetricsValue false when status is targetSolution but agent has no supported metric type`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "targetSolution",
    )

    every {
        userInfoProvider.currentUser()
    } returns cmsAdminUser()

    mockAgentRepository(
        agents = listOf(agent),
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes,
        )
    } returns emptySet()

    val result = service.getAIAgentList(request)

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].hasMetricsValue).isFalse()

    verify(exactly = 1) {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes,
        )
    }
}

Test
fun `getAIAgentList should return hasMetricsValue false when metric type exists but status is not targetSolution`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "implementation",
    )

    every {
        userInfoProvider.currentUser()
    } returns cmsAdminUser()

    mockAgentRepository(
        agents = listOf(agent),
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes,
        )
    } returns setOf(1L)

    val result = service.getAIAgentList(request)

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].hasMetricsValue).isFalse()

    verify(exactly = 1) {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes,
        )
    }
}

@Test
fun `getAIAgentList should return hasMetricsValue false when agent status is null`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
    )

    val agent = createAgent(
        id = 1L,
        statusCode = null,
    )

    every {
        userInfoProvider.currentUser()
    } returns cmsAdminUser()

    mockAgentRepository(
        agents = listOf(agent),
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes,
        )
    } returns setOf(1L)

    val result = service.getAIAgentList(request)

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].hasMetricsValue).isFalse()
}

@Test
fun `getAIAgentList should calculate hasMetricsValue separately for every agent`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
    )

    val targetSolutionAgent = createAgent(
        id = 1L,
        statusCode = "targetSolution",
    )

    val anotherStatusAgent = createAgent(
        id = 2L,
        statusCode = "implementation",
    )

    every {
        userInfoProvider.currentUser()
    } returns cmsAdminUser()

    mockAgentRepository(
        agents = listOf(
            targetSolutionAgent,
            anotherStatusAgent,
        ),
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L, 2L),
            agentTypes = metricAgentTypes,
        )
    } returns setOf(1L, 2L)

    val result = service.getAIAgentList(request)

    Assertions.assertThat(result.content).hasSize(2)

    Assertions.assertThat(
        result.content.first { it.id == 1L }.hasMetricsValue,
    ).isTrue()

    Assertions.assertThat(
        result.content.first { it.id == 2L }.hasMetricsValue,
    ).isFalse()
}

@Test
fun `getAIAgentList should not call initiative metric type repository when agents page is empty`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
    )

    every {
        userInfoProvider.currentUser()
    } returns cmsAdminUser()

    mockAgentRepository(
        agents = emptyList(),
    )

    val result = service.getAIAgentList(request)

    Assertions.assertThat(result.content).isEmpty()

    verify(exactly = 0) {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = any(),
            agentTypes = any(),
        )
    }
}

private val metricAgentTypes = setOf(
    InitiativeMetricAgentType.autonomous.value,
    InitiativeMetricAgentType.copilot.value,
)

private fun cmsAdminUser() = UserDto(
    id = 10L,
    roles = setOf("CMS_ADMIN"),
    email = null,
    login = null,
    firstName = null,
    lastName = null,
    patronymic = null,
    phoneNumber = null,
    position = null,
    sberbankEmployee = null,
    companyId = null,
)

private fun createAgent(
    id: Long = 1L,
    statusCode: String? = null,
): AIAgentEntity {

    val status = statusCode?.let { code ->
        val statusEntity = mockk<StatusEntity>(relaxed = true)

        every {
            statusEntity.code
        } returns code

        statusEntity
    }

    return AIAgentEntity(
        agentId = "agent-$id",
        agentName = "Agent $id",
        disabled = false,
    ).apply {
        this.id = id
        this.agentStatus = status
    }
}

private fun mockAgentRepository(
    agents: List<AIAgentEntity>,
) {
    every {
        aiAgentRepository.findAll(
            search = any(),
            terbankIds = any(),
            programCodes = any(),
            agentStatusCodes = any(),
            blockCodes = any(),
            divisionCodes = any(),
            platformCodes = any(),
            initiativeTypes = any(),
            userAudiences = any(),
            disabled = any(),
            deadlineExpiredIds = any(),
            userId = any(),
            sortField = any(),
            sortDirection = any(),
            pageRequest = any(),
        )
    } returns PageImpl(agents)
}

```
