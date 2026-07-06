```java
@Test
fun `getAIAgentList should return hasMetricsValue true when agent has initiative metric type`() {
    val user = UserDto(
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
        companyId = null
    )

    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val agent = AIAgentEntity(
        agentId = "agent-1",
        agentName = "Agent 1",
        disabled = false
    ).apply {
        id = 1L
    }

    every { userInfoProvider.currentUser() } returns user

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
            pageRequest = any()
        )
    } returns PageImpl(listOf(agent))

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L)
        )
    } returns setOf(1L)

    val result = service.getAIAgentList(request)

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].hasMetricsValue).isTrue()

    verify(exactly = 1) {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L)
        )
    }
}

@Test
fun `getAIAgentList should return hasMetricsValue false when agent has no initiative metric type`() {
    val user = UserDto(
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
        companyId = null
    )

    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val agent = AIAgentEntity(
        agentId = "agent-1",
        agentName = "Agent 1",
        disabled = false
    ).apply {
        id = 1L
    }

    every { userInfoProvider.currentUser() } returns user

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
            pageRequest = any()
        )
    } returns PageImpl(listOf(agent))

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L)
        )
    } returns emptySet()

    val result = service.getAIAgentList(request)

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].hasMetricsValue).isFalse()

    verify(exactly = 1) {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L)
        )
    }
}

@Test
fun `getAIAgentList should not call initiative metric type repository when agents page is empty`() {
    val user = UserDto(
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
        companyId = null
    )

    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    every { userInfoProvider.currentUser() } returns user

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
            pageRequest = any()
        )
    } returns PageImpl(emptyList())

    val result = service.getAIAgentList(request)

    Assertions.assertThat(result.content).isEmpty()

    verify(exactly = 0) {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(any())
    }
}

```
