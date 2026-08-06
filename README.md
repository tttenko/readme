```java

@Test
fun `getAIAgentList should filter page by initiatives with deviation`() {
    val user = cmsAdminUser()

    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
        userId = user.id
    )

    val agent = createAgent(
        id = 2L,
        statusCode = "implementation"
    )

    every { userInfoProvider.currentUser() } returns user

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
        initiativeDeviationCalculator.calculate(
            initiativeIds = setOf(2L)
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
            userId = user.id,
            deadlineExpiredIds = any(),
            sortField = any(),
            sortDirection = any(),
            filterByDeviation = true,
            deviationInitiativeIds = setOf(2L, 3L),
            pageRequest = any()
        )
    }
}
2. Без userId отклонения рассчитывает TRANSFORMATION_OFFICE
@Test
fun `getAIAgentList should return empty page when hasDeviation is true and no deviations found`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    every {
        userInfoProvider.currentUser()
    } returns transformationOfficeUser()

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

    verify(exactly = 1) {
        initiativeDeviationCalculator.findInitiativeIdsWithAnyDeviation(
            initiativeIds = setOf(1L, 2L)
        )
    }

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
        initiativeDeviationCalculator.calculate(
            initiativeIds = any()
        )
    }
}
3. TRANSFORMATION_OFFICE без userId получает рассчитанные отклонения
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

    every {
        userInfoProvider.currentUser()
    } returns transformationOfficeUser()

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

    verify(exactly = 1) {
        initiativeDeviationCalculator.calculate(
            initiativeIds = setOf(1L, 2L)
        )
    }

    verify(exactly = 0) {
        initiativeDeviationCalculator.findInitiativeIdsWithAnyDeviation(
            initiativeIds = any()
        )
    }
}

Здесь дополнительно проверяется важный момент: hasDeviation=false отключает только фильтрацию, но не заполнение поля deviations, когда роль разрешает расчёт.

4. При пустой странице calculate больше не вызывается
@Test
fun `getAIAgentList should not call metric type repository when agents page is empty`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    every {
        userInfoProvider.currentUser()
    } returns transformationOfficeUser()

    mockAgentRepository(
        agents = emptyList()
    )

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

    verify(exactly = 0) {
        initiativeDeviationCalculator.calculate(
            initiativeIds = any()
        )
    }

    verify(exactly = 0) {
        initiativeDeviationCalculator.findInitiativeIdsWithAnyDeviation(
            initiativeIds = any()
        )
    }
}

Предыдущая проверка:

verify(exactly = 1) {
    initiativeDeviationCalculator.calculate(emptySet())
}

теперь некорректна, потому что в новом коде есть проверка:

if (calculateDeviations && initiativeIds.isNotEmpty())
Новые необходимые тесты
5. PROJECT_OFFICE с userId рассчитывает отклонения
@Test
fun `getAIAgentList should calculate deviations for project office when userId is specified`() {
    val user = projectOfficeUser()

    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
        userId = user.id
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "implementation"
    )

    val deviation = InitiativeDeviationResponse(
        code = InitiativeDeviationCode.STAGE_DEADLINE_EXPIRED.name,
        priority = 1,
        title = "Нарушен дедлайн этапа",
        description = "Срок завершения этапа уже прошёл"
    )

    every {
        userInfoProvider.currentUser()
    } returns user

    mockAgentRepository(
        agents = listOf(agent)
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes
        )
    } returns emptySet()

    every {
        initiativeDeviationCalculator.calculate(
            initiativeIds = setOf(1L)
        )
    } returns mapOf(
        1L to listOf(deviation)
    )

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = false
    )

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content.first().deviations)
        .containsExactly(deviation)

    verify(exactly = 1) {
        initiativeDeviationCalculator.calculate(
            initiativeIds = setOf(1L)
        )
    }
}
6. TRANSFORMATION_OFFICE с userId не рассчитывает отклонения
@Test
fun `getAIAgentList should not calculate deviations for transformation office when userId is specified`() {
    val user = transformationOfficeUser()

    val request = ShowcaseRequestDto(
        page = 0,
        size = 10,
        userId = user.id
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "implementation"
    )

    every {
        userInfoProvider.currentUser()
    } returns user

    mockAgentRepository(
        agents = listOf(agent)
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes
        )
    } returns emptySet()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = true
    )

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content.first().deviations).isEmpty()

    verifyDeviationCalculationWasNotPerformed()

    verifyFindAllWithoutDeviationFilter(
        expectedUserId = user.id
    )
}

Этот тест также проверяет, что hasDeviation=true игнорируется, когда роль не имеет права рассчитывать отклонения.

7. CMS_ADMIN без userId не рассчитывает отклонения
@Test
fun `getAIAgentList should not calculate deviations for cms admin when userId is not specified`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "implementation"
    )

    every {
        userInfoProvider.currentUser()
    } returns cmsAdminUser()

    mockAgentRepository(
        agents = listOf(agent)
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes
        )
    } returns emptySet()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = true
    )

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content.first().deviations).isEmpty()

    verifyDeviationCalculationWasNotPerformed()

    verifyFindAllWithoutDeviationFilter(
        expectedUserId = null
    )
}
8. PROJECT_OFFICE без userId не рассчитывает отклонения
@Test
fun `getAIAgentList should not calculate deviations for project office when userId is not specified`() {
    val request = ShowcaseRequestDto(
        page = 0,
        size = 10
    )

    val agent = createAgent(
        id = 1L,
        statusCode = "implementation"
    )

    every {
        userInfoProvider.currentUser()
    } returns projectOfficeUser()

    mockAgentRepository(
        agents = listOf(agent)
    )

    every {
        initiativeMetricTypeRepository.findInitiativeIdsWithAgentTypes(
            initiativeIds = setOf(1L),
            agentTypes = metricAgentTypes
        )
    } returns emptySet()

    val result = service.getAIAgentList(
        showcaseRequestDto = request,
        hasDeviation = true
    )

    Assertions.assertThat(result.content).hasSize(1)
    Assertions.assertThat(result.content.first().deviations).isEmpty()

    verifyDeviationCalculationWasNotPerformed()

    verifyFindAllWithoutDeviationFilter(
        expectedUserId = null
    )
}
Вспомогательные методы для новых тестов

Тип expectedUserId укажи такой же, как в ShowcaseRequestDto.

private fun verifyDeviationCalculationWasNotPerformed() {
    verify(exactly = 0) {
        initiativeDeviationCalculator.findInitiativeIdsWithAnyDeviation(
            initiativeIds = any()
        )
    }

    verify(exactly = 0) {
        initiativeDeviationCalculator.calculate(
            initiativeIds = any()
        )
    }

    verify(exactly = 0) {
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
    }
}
private fun verifyFindAllWithoutDeviationFilter(
    expectedUserId: String?
) {
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
            userId = expectedUserId,
            deadlineExpiredIds = any(),
            sortField = any(),
            sortDirection = any(),
            filterByDeviation = false,
            deviationInitiativeIds = setOf(-1L),
            pageRequest = any()
        )
    }
}

```
