```java

@DisplayName("InitiativeRegistryExportService tests")
class InitiativeRegistryExportServiceTest {

    private lateinit var terBankService: TerBankService
    private lateinit var aiAgentRepository: AIAgentRepository
    private lateinit var statusRepository: StatusRepository
    private lateinit var agentStatusSlaRepository: AgentStatusSlaRepository
    private lateinit var authFeignClient: PrmAuthFeignClient
    private lateinit var financialEffectRepository: InitiativeFinancialEffectRepository
    private lateinit var service: InitiativeRegistryExportService

    private val objectMapper = ObjectMapper()

    @BeforeEach
    fun setUp() {
        terBankService = mockk()
        aiAgentRepository = mockk()
        statusRepository = mockk()
        agentStatusSlaRepository = mockk()
        authFeignClient = mockk()
        financialEffectRepository = mockk()

        service = InitiativeRegistryExportService(
            mapper = InitiativeRegistryMapper(),
            terBankService = terBankService,
            aiAgentRepository = aiAgentRepository,
            statusRepository = statusRepository,
            agentStatusSlaRepository = agentStatusSlaRepository,
            authFeignClient = authFeignClient,
            financialEffectRepository = financialEffectRepository
        )

        ReflectionUtils.setField(service, "portalUrl", "https://pult.sber.ru/")
    }

    private fun status(code: String, ordering: Long) = StatusEntity().also {
        it.id = ordering
        it.code = code
        it.name = code
        it.ordering = ordering
    }

    private fun sla(statusCode: String, plannedDate: LocalDateTime? = null, initiativeId: Long? = 1L) =
        AgentStatusSlaEntity().also {
            it.primaryKey.aiAgentId = initiativeId
            it.agentStatus = status(statusCode, 1L)
            it.plannedDate = plannedDate
        }

    private fun agent(id: Long) = AIAgentEntity().also {
        it.id = id
        it.agentId = "AI-$id"
        it.agentName = "Инициатива $id"
    }

    private fun financialEffect(
        id: Long,
        initiativeId: Long,
        effectYear: Int = 2026,
        plan: String = "1000",
        fact: String? = "1200",
        targetMonth: LocalDate = YearMonth.of(2026, 3).atDay(1),
        code: String = "revenue",
        typeName: String = "Выручка",
        unit: String = "руб.",
        approvalLinks: List<String> = listOf("https://jira.example.com/PRM-1"),
    ): InitiativeFinancialEffectEntity {
        val type = FinancialEffectTypeEntity(
            code = code,
            name = typeName,
            unit = unit,
            moreIsBetter = true,
        )
        return InitiativeFinancialEffectEntity(
            initiative = AIAgentEntity().apply { this.id = initiativeId },
            effectYear = effectYear,
            type = type,
            planValue = BigDecimal(plan),
            factValue = fact?.let { BigDecimal(it) },
            targetMonth = targetMonth,
            approvalLinks = objectMapper.valueToTree(approvalLinks),
            disabled = false,
            createdBy = 1L,
        ).apply {
            this.id = id
            this.created = LocalDateTime.of(effectYear, 1, 1, 0, 0)
        }
    }

    private fun mockExportDependencies(agents: List<AIAgentEntity> = listOf(agent(1L))) {
        every { terBankService.list() } returns emptyList()
        every { aiAgentRepository.findAll() } returns agents
        every { statusRepository.findAllActive() } returns emptyList()
        every { agentStatusSlaRepository.findAllByAiAgentIdIn(any()) } returns emptyList()
        every { financialEffectRepository.findActiveByInitiativeIds(any()) } returns emptyList()
    }
```
