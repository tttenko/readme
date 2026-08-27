```java

import io.mockk.MockK
import io.mockk.every
import io.mockk.junit5.MockKExtension
import io.mockk.verify
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.ExtendWith
import ru.sber.prm.repository.AIAgentRepository
import ru.sber.prm.repository.AgentStrategyRepository
import ru.sber.prm.repository.InvolvedResourceRepository

@ExtendWith(MockKExtension::class)
class JiraNewInitiativeCreationServiceTest {

    @MockK
    private lateinit var agentRepository: AIAgentRepository

    @MockK
    private lateinit var agentStrategyRepository: AgentStrategyRepository

    @MockK
    private lateinit var involvedResourceRepository: InvolvedResourceRepository

    @MockK
    private lateinit var organizationResolver: JiraInitiativeOrganizationResolver

    @MockK
    private lateinit var initiativeTypeResolver: JiraInitiativeTypeResolver

    @MockK
    private lateinit var numericValueParser: JiraNumericValueParser

    @MockK
    private lateinit var strategyResolver: JiraStrategyResolver

    @MockK
    private lateinit var involvedResourceResolver: JiraInvolvedResourceResolver

    @MockK
    private lateinit var contactCreator: JiraInitiativeContactCreator

    @MockK
    private lateinit var enablerCreator: JiraInitiativeEnablerCreator

    @MockK
    private lateinit var qualityGateCreator: JiraInitiativeQualityGateCreator

    @MockK
    private lateinit var issueRelationCreator: JiraInitiativeIssueRelationCreator

    private lateinit var service: JiraNewInitiativeCreationService

    @BeforeEach
    fun setUp() {
        service = JiraNewInitiativeCreationService(
            agentRepository = agentRepository,
            agentStrategyRepository = agentStrategyRepository,
            involvedResourceRepository = involvedResourceRepository,
            organizationResolver = organizationResolver,
            initiativeTypeResolver = initiativeTypeResolver,
            numericValueParser = numericValueParser,
            strategyResolver = strategyResolver,
            involvedResourceResolver = involvedResourceResolver,
            contactCreator = contactCreator,
            enablerCreator = enablerCreator,
            qualityGateCreator = qualityGateCreator,
            issueRelationCreator = issueRelationCreator,
        )
    }

    @Test
    fun `should skip initiative when block and division cannot be resolved`() {
        val initiatorUnits = listOf("UNKNOWN_INITIATOR")
        val executorUnits = listOf("UNKNOWN_EXECUTOR")

        val fields = mockk<SearchIssueFieldsDto> {
            every { customfield_30000 } returns initiatorUnits
            every { customfield_30001 } returns executorUnits
        }

        val issue = mockk<SearchIssueDto> {
            every { key } returns JIRA_KEY
            every { this@mockk.fields } returns fields
        }

        val referenceData = JiraImportReferenceData(
            strategiesByJiraKey = emptyMap(),
            enablersByNormalizedName = emptyMap(),
            statusesByCode = emptyMap(),
            qualityGates = emptyList(),
            divisionsByLabel = emptyMap(),
            blocksByLabel = emptyMap(),
            initiativeTypesByCode = emptyMap(),
        )

        every {
            organizationResolver.resolveOrganization(
                initiatorUnits = initiatorUnits,
                executorUnits = executorUnits,
                referenceData = referenceData,
            )
        } returns JiraInitiativeOrganization(
            division = null,
            block = null,
        )

        val result = service.createInitiativeFromJira(
            issue = issue,
            referenceData = referenceData,
        )

        assertNull(result)

        verify(exactly = 1) {
            organizationResolver.resolveOrganization(
                initiatorUnits = initiatorUnits,
                executorUnits = executorUnits,
                referenceData = referenceData,
            )
        }

        verify(exactly = 0) {
            agentRepository.save(any())
        }

        verify(exactly = 0) {
            initiativeTypeResolver.resolveInitiativeType(
                labels = any(),
                initiativeTypesByCode = any(),
            )
        }

        verify(exactly = 0) {
            contactCreator.createContacts(
                agent = any(),
                issue = any(),
            )
        }

        verify(exactly = 0) {
            enablerCreator.createEnablers(
                agent = any(),
                issue = any(),
                referenceData = any(),
            )
        }

        verify(exactly = 0) {
            qualityGateCreator.createQualityGates(
                agent = any(),
                referenceData = any(),
                currentDateTime = any(),
            )
        }

        verify(exactly = 0) {
            issueRelationCreator.createIssueRelations(
                agent = any(),
                issue = any(),
                currentDateTime = any(),
            )
        }
    }

    private companion object {
        const val JIRA_KEY = "CROSSGOAL-300"
    }
}
```
