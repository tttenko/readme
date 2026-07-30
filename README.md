```java

@Test
fun `getAIAgentList should throw forbidden when non admin requests another userId`() {
    val user = UserDto(
        id = 10L,
        roles = setOf("PROJECT_OFFICE"),
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
        size = 10,
        userId = 99L
    )

    every { userInfoProvider.currentUser() } returns user
    every {
        messageProvider[
            Metadata.ErrorMessages.FORBIDDEN_USER_ID_FILTER
        ]
    } returns "forbidden"

    val exception = assertThrows<AiForbiddenException> {
        service.getAIAgentList(
            showcaseRequestDto = request,
            hasDeviation = false
        )
    }

    assertEquals(
        Metadata.ErrorMessages.FORBIDDEN_USER_ID_FILTER,
        exception.errorCode
    )

    verify(exactly = 0) {
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
            userId = any(),
            deadlineExpiredIds = any(),
            sortField = any(),
            sortDirection = any(),
            filterByDeviation = any(),
            deviationInitiativeIds = any(),
            pageRequest = any()
        )
    }

    verify(exactly = 0) {
        initiativeDeviationCalculator.calculate(any())
    }
}
@Test
fun `getAIAgentList should call repository with deadlineExpired sort`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
        sort = AiAgentsSort.deadlineExpired
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(agents = emptyList())

    every {
        initiativeDeviationCalculator.calculate(emptySet())
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content).isEmpty()

    verify(exactly = 1) {
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
            userId = any(),
            deadlineExpiredIds = any(),
            sortField = "deadline_priority",
            sortDirection = "DESC",
            filterByDeviation = false,
            deviationInitiativeIds = setOf(-1L),
            pageRequest = any()
        )
    }
}
@Test
fun `getAIAgentList should call repository with alphabet sort`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
        sort = AiAgentsSort.alphabet,
        sortType = Sort.Direction.ASC
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(agents = emptyList())

    every {
        initiativeDeviationCalculator.calculate(emptySet())
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content).isEmpty()

    verify(exactly = 1) {
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
            userId = any(),
            deadlineExpiredIds = any(),
            sortField = "alphabet",
            sortDirection = "ASC",
            filterByDeviation = false,
            deviationInitiativeIds = setOf(-1L),
            pageRequest = any()
        )
    }
}
Исправленные тесты hasMetricsValue
@Test
fun `getAIAgentList should return hasMetricsValue true when status is targetSolution and agent has supported metric type`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "targetSolution"
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(agents = listOf(agent))

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes
        )
    } returns setOf(1L)

    every {
        initiativeDeviationCalculator.calculate(
            initiativeIds = setOf(1L)
        )
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].hasMetricsValue).isTrue()

    verify(exactly = 1) {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes
        )
    }
}
@Test
fun `getAIAgentList should return hasMetricsValue false when status is targetSolution but agent has no supported metric type`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "targetSolution"
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(agents = listOf(agent))

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes
        )
    } returns emptySet()

    every {
        initiativeDeviationCalculator.calculate(setOf(1L))
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].hasMetricsValue).isFalse()
}
@Test
fun `getAIAgentList should return hasMetricsValue false when metric type exists but status is not targetSolution`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "implementation"
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(agents = listOf(agent))

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes
        )
    } returns setOf(1L)

    every {
        initiativeDeviationCalculator.calculate(setOf(1L))
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].hasMetricsValue).isFalse()
}
@Test
fun `getAIAgentList should return hasMetricsValue false when agent status is null`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val agent = createAgent(
        id = 1L,
        statusCode = null
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(agents = listOf(agent))

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes
        )
    } returns setOf(1L)

    every {
        initiativeDeviationCalculator.calculate(setOf(1L))
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].hasMetricsValue).isFalse()
}
@Test
fun `getAIAgentList should calculate hasMetricsValue separately for every agent`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val targetSolutionAgent = createAgent(
        id = 1L,
        statusCode = "targetSolution"
    )

    val anotherStatusAgent = createAgent(
        id = 2L,
        statusCode = "implementation"
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(
        agents = listOf(
            targetSolutionAgent,
            anotherStatusAgent
        )
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L, 2L),
            agentTypes = metricAgentTypes
        )
    } returns setOf(1L, 2L)

    every {
        initiativeDeviationCalculator.calculate(
            setOf(1L, 2L)
        )
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content).hasSize(2)

    Assertions.assertThat(
        result.content.first { it.id == 1L }.hasMetricsValue
    ).isTrue()

    Assertions.assertThat(
        result.content.first { it.id == 2L }.hasMetricsValue
    ).isFalse()
}
Пустая страница
@Test
fun `getAIAgentList should not call metric type repository when agents page is empty`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(agents = emptyList())

    every {
        initiativeDeviationCalculator.calculate(emptySet())
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content).isEmpty()

    verify(exactly = 0) {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = any(),
            agentTypes = any()
        )
    }

    verify(exactly = 1) {
        initiativeDeviationCalculator.calculate(emptySet())
    }
}
Новые тесты для hasDeviation
hasDeviation=false не запускает предварительный поиск
@Test
fun `getAIAgentList should not filter initiatives by deviation when hasDeviation is false`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(agents = emptyList())

    every {
        initiativeDeviationCalculator.calculate(emptySet())
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content).isEmpty()

    verify(exactly = 0) {
        initiativeDeviationCalculator
            .findInitiativeIdsWithAnyDeviation(any())
    }

    verify(exactly = 1) {
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
            userId = any(),
            deadlineExpiredIds = any(),
            sortField = any(),
            sortDirection = any(),
            filterByDeviation = false,
            deviationInitiativeIds = setOf(-1L),
            pageRequest = any()
        )
    }
}
hasDeviation=true передаёт найденные ID в основной запрос
@Test
fun `getAIAgentList should filter page by initiatives with deviation`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val agent = createAgent(
        id = 2L,
        statusCode = "implementation"
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockFilteredInitiativeIds(
        initiativeIds = setOf(1L, 2L, 3L)
    )

    every {
        initiativeDeviationCalculator.findInitiativeIdsWithAnyDeviation(
            initiativeIds = setOf(1L, 2L, 3L)
        )
    } returns setOf(2L, 3L)

    mockAgentRepository(
        agents = listOf(agent),
        filterByDeviation = true,
        deviationInitiativeIds = setOf(2L, 3L)
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(2L),
            agentTypes = metricAgentTypes
        )
    } returns emptySet()

    every {
        initiativeDeviationCalculator.calculate(
            initiativeIds = setOf(2L)
        )
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = true
    )

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content[0].id).isEqualTo(2L)

    verify(exactly = 1) {
        initiativeDeviationCalculator.findInitiativeIdsWithAnyDeviation(
            initiativeIds = setOf(1L, 2L, 3L)
        )
    }

    verify(exactly = 1) {
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
            userId = any(),
            deadlineExpiredIds = any(),
            sortField = any(),
            sortDirection = any(),
            filterByDeviation = true,
            deviationInitiativeIds = setOf(2L, 3L),
            pageRequest = any()
        )
    }
}
Нет ни одной инициативы с отклонением
@Test
fun `getAIAgentList should return empty page when hasDeviation is true and no deviations found`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockFilteredInitiativeIds(
        initiativeIds = setOf(1L, 2L)
    )

    every {
        initiativeDeviationCalculator.findInitiativeIdsWithAnyDeviation(
            initiativeIds = setOf(1L, 2L)
        )
    } returns emptySet()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = true
    )

    Assertions.assertThat(result.content).isEmpty()
    Assertions.assertThat(result.totalElements).isZero()
    Assertions.assertThat(result.totalPages).isZero()

    verify(exactly = 0) {
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
            userId = any(),
            deadlineExpiredIds = any(),
            sortField = any(),
            sortDirection = any(),
            filterByDeviation = any(),
            deviationInitiativeIds = any(),
            pageRequest = any()
        )
    }

    verify(exactly = 0) {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = any(),
            agentTypes = any()
        )
    }

    verify(exactly = 0) {
        initiativeDeviationCalculator.calculate(any())
    }
}
Проверка добавления отклонений в ответ
@Test
fun `getAIAgentList should add calculated deviations to corresponding agent`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val firstAgent = createAgent(
        id = 1L,
        statusCode = "implementation"
    )

    val secondAgent = createAgent(
        id = 2L,
        statusCode = "implementation"
    )

    val deviation = InitiativeDeviationResponse(
        code = InitiativeDeviationCode.STAGE_DEADLINE_EXPIRED.name,
        priority = 1,
        title = "Нарушен дедлайн этапа",
        description = "Срок завершения этапа уже прошёл"
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(
        agents = listOf(firstAgent, secondAgent)
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L, 2L),
            agentTypes = metricAgentTypes
        )
    } returns emptySet()

    every {
        initiativeDeviationCalculator.calculate(
            initiativeIds = setOf(1L, 2L)
        )
    } returns mapOf(
        1L to listOf(deviation),
        2L to emptyList()
    )

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    val firstResponse = result.content.first { it.id == 1L }
    val secondResponse = result.content.first { it.id == 2L }

    Assertions.assertThat(firstResponse.deviations)
        .containsExactly(deviation)

    Assertions.assertThat(secondResponse.deviations)
        .isEmpty()
}
Если калькулятор не вернул ключ инициативы

Этот тест проверяет .orEmpty():

@Test
fun `getAIAgentList should return empty deviations when calculator has no entry for agent`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "implementation"
    )

    every { userInfoProvider.currentUser() } returns cmsAdminUser()

    mockAgentRepository(agents = listOf(agent))

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes
        )
    } returns emptySet()

    every {
        initiativeDeviationCalculator.calculate(setOf(1L))
    } returns emptyMap()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content.single().deviations).isEmpty()
}
Обновлённые вспомогательные методы
private fun mockAgentRepository(
    agents: List<AIAgentEntity>,
    filterByDeviation: Boolean = false,
    deviationInitiativeIds: Set<Long> = setOf(-1L)
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
            userId = any(),
            deadlineExpiredIds = any(),
            sortField = any(),
            sortDirection = any(),
            filterByDeviation = filterByDeviation,
            deviationInitiativeIds = deviationInitiativeIds,
            pageRequest = any()
        )
    } returns PageImpl(agents)
}
private fun mockFilteredInitiativeIds(
    initiativeIds: Set<Long>
) {
    every {
        aiAgentRepository.findFilteredInitiativeIds(
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
            userId = any(),
            deadlineExpiredIds = any()
        )
    } returns initiativeIds
}

```
