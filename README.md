```java
@ExtendWith(MockKExtension::class)
internal class MetricsServiceTest {

    @MockK
    private lateinit var metricsDirectoryRepository: MetricsDirectoryRepository

    @MockK
    private lateinit var initiativeMetricValueRepository: InitiativeMetricValueRepository

    @MockK
    private lateinit var messageProvider: MessageProvider

    @MockK
    private lateinit var userInfoProvider: UserInfoProvider

    @MockK
    private lateinit var authFeignClient: PrmAuthFeignClient

    @MockK
    private lateinit var dateTimeProvider: DateTimeProvider

    private lateinit var service: MetricsService

    @BeforeEach
    fun setUp() {
        service = MetricsService(
            metricsDirectoryRepository = metricsDirectoryRepository,
            initiativeMetricValueRepository = initiativeMetricValueRepository,
            messageProvider = messageProvider,
            userInfoProvider = userInfoProvider,
            authFeignClient = authFeignClient,
            dateTimeProvider = dateTimeProvider,
        )
    }

    @Test
    fun `createMetric should create metric with success`() {
        // given
        val request = CreateMetricRequest(
            name = "Количество обращений",
            unit = "шт.",
            direction = "Рост",
            agentTypes = setOf(
                MetricAgentType.AUTONOMOUS,
                MetricAgentType.COPILOT,
            ),
            description = "Описание метрики",
            frequency = "Ежемесячно",
        )

        mockCurrentUser()
        every {
            dateTimeProvider.currentDateTime()
        } returns CURRENT_DATE_TIME

        every {
            metricsDirectoryRepository.save(any())
        } answers {
            firstArg<MetricsDirectoryEntity>().apply {
                id = METRIC_ID
            }
        }

        // when
        val result = service.createMetric(request = request)

        // then
        assertEquals(METRIC_ID, result.id)
        assertEquals("Количество обращений", result.name)
        assertEquals("шт.", result.unit)
        assertEquals("Рост", result.direction)
        assertEquals(true, result.isActive)
        assertEquals("Описание метрики", result.description)
        assertEquals("Ежемесячно", result.frequency)
        assertEquals(true, result.canBeDeleted)
        assertEquals(CURRENT_DATE_TIME, result.lastModifiedAt)

        verify(exactly = 1) {
            metricsDirectoryRepository.save(
                match { metric ->
                    metric.name == request.name &&
                        metric.unit == request.unit &&
                        metric.direction == request.direction &&
                        metric.description == request.description &&
                        metric.frequency == request.frequency &&
                        metric.autonomousApplicability == true &&
                        metric.copilotApplicability == true &&
                        metric.requiresAppealsWork == false &&
                        metric.active == true &&
                        metric.updatedBy == USER_ID &&
                        metric.updatedAt == CURRENT_DATE_TIME
                },
            )
        }
    }

    @Test
    fun `createMetric should throw bad request when agent types are wrong`() {
        // given
        val request = CreateMetricRequest(
            name = "Количество обращений",
            unit = "шт.",
            direction = "Рост",
            agentTypes = setOf("wrong_type"),
            description = "Описание метрики",
            frequency = "Ежемесячно",
        )

        every {
            messageProvider[WRONG_METRIC_AGENT_TYPES]
        } returns "Некорректные режимы работы метрики: {0}"

        // when / then
        val exception = assertThrows<AiBadRequestException> {
            service.createMetric(request = request)
        }

        assertEquals(WRONG_METRIC_AGENT_TYPES, exception.errorCode)

        verify(exactly = 0) {
            userInfoProvider.currentUser()
        }

        verify(exactly = 0) {
            metricsDirectoryRepository.save(any())
        }
    }

    @Test
    fun `getMetrics should return empty list when metrics not found`() {
        // given
        every {
            metricsDirectoryRepository.findAll()
        } returns emptyList()

        // when
        val result = service.getMetrics()

        // then
        assertTrue(result.isEmpty())

        verify(exactly = 1) {
            metricsDirectoryRepository.findAll()
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository.findUsedMetricDirectoryIds(any())
        }

        verify(exactly = 0) {
            authFeignClient.getUsers(any(), any())
        }
    }

    @Test
    fun `getMetrics should return metrics with canBeDeleted values`() {
        // given
        val usedMetricId = UUID.randomUUID()
        val unusedMetricId = UUID.randomUUID()

        val usedMetric = metricEntity(
            id = usedMetricId,
            name = "Используемая метрика",
            updatedBy = USER_ID,
        )

        val unusedMetric = metricEntity(
            id = unusedMetricId,
            name = "Неиспользуемая метрика",
            updatedBy = USER_ID,
        )

        every {
            metricsDirectoryRepository.findAll()
        } returns listOf(usedMetric, unusedMetric)

        every {
            initiativeMetricValueRepository.findUsedMetricDirectoryIds(
                metricDirectoryIds = setOf(usedMetricId, unusedMetricId),
            )
        } returns setOf(usedMetricId)

        mockAuthUsers(userIds = setOf(USER_ID))

        // when
        val result = service.getMetrics()

        // then
        assertEquals(2, result.size)

        assertEquals(usedMetricId, result[0].id)
        assertEquals("Используемая метрика", result[0].name)
        assertEquals(false, result[0].canBeDeleted)

        assertEquals(unusedMetricId, result[1].id)
        assertEquals("Неиспользуемая метрика", result[1].name)
        assertEquals(true, result[1].canBeDeleted)

        verify(exactly = 1) {
            metricsDirectoryRepository.findAll()
        }

        verify(exactly = 1) {
            initiativeMetricValueRepository.findUsedMetricDirectoryIds(
                metricDirectoryIds = setOf(usedMetricId, unusedMetricId),
            )
        }

        verify(exactly = 1) {
            authFeignClient.getUsers(
                ids = setOf(USER_ID),
                companyId = null,
            )
        }
    }

    @Test
    fun `updateMetricActivity should update active flag with success`() {
        // given
        val metric = metricEntity(
            id = METRIC_ID,
            active = true,
        )

        val request = UpdateMetricActivityRequest(
            active = false,
        )

        mockCurrentUser()

        every {
            metricsDirectoryRepository.findById(METRIC_ID)
        } returns Optional.of(metric)

        every {
            dateTimeProvider.currentDateTime()
        } returns CURRENT_DATE_TIME

        every {
            metricsDirectoryRepository.save(any())
        } answers {
            firstArg()
        }

        every {
            initiativeMetricValueRepository.countByMetricDirectoryId(
                metricDirectoryId = METRIC_ID,
            )
        } returns 1L

        // when
        val result = service.updateMetricActivity(
            metricId = METRIC_ID,
            request = request,
        )

        // then
        assertEquals(METRIC_ID, result.id)
        assertEquals(false, result.isActive)
        assertEquals(false, result.canBeDeleted)
        assertEquals(CURRENT_DATE_TIME, result.lastModifiedAt)

        verify(exactly = 1) {
            metricsDirectoryRepository.save(
                match { savedMetric ->
                    savedMetric.id == METRIC_ID &&
                        savedMetric.active == false &&
                        savedMetric.updatedBy == USER_ID &&
                        savedMetric.updatedAt == CURRENT_DATE_TIME
                },
            )
        }
    }

    @Test
    fun `updateMetricActivity should throw not found when metric not found`() {
        // given
        val request = UpdateMetricActivityRequest(
            active = false,
        )

        mockCurrentUser()

        every {
            metricsDirectoryRepository.findById(METRIC_ID)
        } returns Optional.empty()

        every {
            messageProvider[METRIC_NOT_FOUND]
        } returns "Метрика с id {0} не найдена"

        // when / then
        val exception = assertThrows<AiNotFoundException> {
            service.updateMetricActivity(
                metricId = METRIC_ID,
                request = request,
            )
        }

        assertEquals(METRIC_NOT_FOUND, exception.errorCode)

        verify(exactly = 0) {
            metricsDirectoryRepository.save(any())
        }

        verify(exactly = 0) {
            initiativeMetricValueRepository.countByMetricDirectoryId(any())
        }
    }

    @Test
    fun `updateMetric should update metric with success`() {
        // given
        val metric = metricEntity(
            id = METRIC_ID,
            name = "Старое название",
            active = true,
        )

        val request = UpdateMetricRequest(
            name = "Новое название",
            unit = "шт.",
            direction = "Рост",
            agentTypes = setOf(
                MetricAgentType.AUTONOMOUS,
                MetricAgentType.COPILOT,
                MetricAgentType.APPEALS,
            ),
            description = "Новое описание",
            frequency = "Ежемесячно",
        )

        mockCurrentUser()

        every {
            metricsDirectoryRepository.findById(METRIC_ID)
        } returns Optional.of(metric)

        every {
            dateTimeProvider.currentDateTime()
        } returns CURRENT_DATE_TIME

        every {
            metricsDirectoryRepository.save(any())
        } answers {
            firstArg()
        }

        every {
            initiativeMetricValueRepository.countByMetricDirectoryId(
                metricDirectoryId = METRIC_ID,
            )
        } returns 0L

        // when
        val result = service.updateMetric(
            metricId = METRIC_ID,
            request = request,
        )

        // then
        assertEquals(METRIC_ID, result.id)
        assertEquals("Новое название", result.name)
        assertEquals("шт.", result.unit)
        assertEquals("Рост", result.direction)
        assertEquals("Новое описание", result.description)
        assertEquals("Ежемесячно", result.frequency)
        assertEquals(true, result.isActive)
        assertEquals(true, result.canBeDeleted)

        verify(exactly = 1) {
            metricsDirectoryRepository.save(
                match { savedMetric ->
                    savedMetric.name == request.name &&
                        savedMetric.unit == request.unit &&
                        savedMetric.direction == request.direction &&
                        savedMetric.description == request.description &&
                        savedMetric.frequency == request.frequency &&
                        savedMetric.autonomousApplicability == true &&
                        savedMetric.copilotApplicability == true &&
                        savedMetric.requiresAppealsWork == true &&
                        savedMetric.active == true &&
                        savedMetric.updatedBy == USER_ID &&
                        savedMetric.updatedAt == CURRENT_DATE_TIME
                },
            )
        }
    }

    @Test
    fun `updateMetric should throw not found when metric not found`() {
        // given
        val request = UpdateMetricRequest(
            name = "Новое название",
            unit = "шт.",
            direction = "Рост",
            agentTypes = setOf(MetricAgentType.AUTONOMOUS),
            description = "Новое описание",
            frequency = "Ежемесячно",
        )

        mockCurrentUser()

        every {
            metricsDirectoryRepository.findById(METRIC_ID)
        } returns Optional.empty()

        every {
            messageProvider[METRIC_NOT_FOUND]
        } returns "Метрика с id {0} не найдена"

        // when / then
        val exception = assertThrows<AiNotFoundException> {
            service.updateMetric(
                metricId = METRIC_ID,
                request = request,
            )
        }

        assertEquals(METRIC_NOT_FOUND, exception.errorCode)

        verify(exactly = 0) {
            metricsDirectoryRepository.save(any())
        }
    }

    @Test
    fun `deleteMetric should delete metric when metric values not found`() {
        // given
        every {
            initiativeMetricValueRepository.countByMetricDirectoryId(
                metricDirectoryId = METRIC_ID,
            )
        } returns 0L

        every {
            metricsDirectoryRepository.deleteByMetricId(
                metricId = METRIC_ID,
            )
        } just runs

        // when
        service.deleteMetric(metricId = METRIC_ID)

        // then
        verify(exactly = 1) {
            initiativeMetricValueRepository.countByMetricDirectoryId(
                metricDirectoryId = METRIC_ID,
            )
        }

        verify(exactly = 1) {
            metricsDirectoryRepository.deleteByMetricId(
                metricId = METRIC_ID,
            )
        }
    }

    @Test
    fun `deleteMetric should throw conflict when metric has values`() {
        // given
        every {
            initiativeMetricValueRepository.countByMetricDirectoryId(
                metricDirectoryId = METRIC_ID,
            )
        } returns 1L

        every {
            messageProvider[METRIC_HAS_VALUES]
        } returns "Метрика не может быть удалена, так как по ней уже сданы значения"

        // when / then
        val exception = assertThrows<ResponseStatusException> {
            service.deleteMetric(metricId = METRIC_ID)
        }

        assertEquals(HttpStatus.CONFLICT, exception.statusCode)

        verify(exactly = 0) {
            metricsDirectoryRepository.deleteByMetricId(any())
        }
    }

    private fun metricEntity(
        id: UUID,
        name: String = "Количество обращений",
        unit: String = "шт.",
        direction: String = "Рост",
        description: String = "Описание метрики",
        frequency: String = "Ежемесячно",
        active: Boolean = true,
        updatedBy: Long = USER_ID,
        updatedAt: LocalDateTime = CURRENT_DATE_TIME,
    ): MetricsDirectoryEntity {
        return MetricsDirectoryEntity().apply {
            this.id = id
            this.name = name
            this.unit = unit
            this.direction = direction
            this.description = description
            this.frequency = frequency
            this.active = active
            this.updatedBy = updatedBy
            this.updatedAt = updatedAt

            this.autonomousApplicability = true
            this.copilotApplicability = false
            this.requiresAppealsWork = false
        }
    }

    private fun mockCurrentUser() {
        every {
            userInfoProvider.currentUser()
        } returns mockk(relaxed = true) {
            every {
                id
            } returns USER_ID
        }
    }

    private fun mockAuthUsers(
        userIds: Set<Long>,
    ) {
        every {
            authFeignClient.getUsers(
                ids = userIds,
                companyId = null,
            )
        } returns ResponseEntity.ok(
            userIds.map { userId ->
                mockk(relaxed = true) {
                    every {
                        id
                    } returns userId
                }
            },
        )
    }

    private companion object {
        private val METRIC_ID: UUID =
            UUID.fromString("11111111-1111-1111-1111-111111111111")

        private const val USER_ID: Long = 100L

        private val CURRENT_DATE_TIME: LocalDateTime =
            LocalDateTime.of(2026, 6, 19, 12, 30)
    }
}
```
