```java
package ru.sber.prm.service

import io.mockk.every
import io.mockk.mockk
import io.mockk.verify
import org.apache.poi.xssf.usermodel.XSSFWorkbook
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Test
import org.springframework.http.ContentDisposition
import org.springframework.http.HttpHeaders
import org.springframework.http.HttpStatus
import ru.sber.prm.config.properties.EmailProperties
import ru.sber.prm.entity.AIAgentEntity
import ru.sber.prm.entity.AgentStatusSlaEntity
import ru.sber.prm.entity.StatusEntity
import ru.sber.prm.exception.AiInternalServerException
import ru.sber.prm.mapper.InitiativeRegistryMapper
import ru.sber.prm.repository.AIAgentRepository
import ru.sber.prm.repository.AgentStatusSlaRepository
import ru.sber.prm.repository.StatusRepository
import ru.sber.prm.service.references.TerBankService
import java.time.LocalDateTime

@DisplayName("InitiativeRegistryExportService tests")
class InitiativeRegistryExportServiceTest {

    private lateinit var terBankService: TerBankService
    private lateinit var aiAgentRepository: AIAgentRepository
    private lateinit var statusRepository: StatusRepository
    private lateinit var agentStatusSlaRepository: AgentStatusSlaRepository
    private lateinit var emailProperties: EmailProperties
    private lateinit var service: InitiativeRegistryExportService

    @BeforeEach
    fun setUp() {
        terBankService = mockk()
        aiAgentRepository = mockk()
        statusRepository = mockk()
        agentStatusSlaRepository = mockk()
        emailProperties = mockk()

        service = InitiativeRegistryExportService(
            mapper = InitiativeRegistryMapper(),
            terBankService = terBankService,
            aiAgentRepository = aiAgentRepository,
            statusRepository = statusRepository,
            agentStatusSlaRepository = agentStatusSlaRepository,
            emailProperties = emailProperties,
        )
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

    @Test
    fun `should use findAll including disabled and load sla in bulk`() {
        // given
        val activeAgent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
            it.agentStatus = status("pilot", 3L)
            it.disabled = false
        }

        val disabledAgent = AIAgentEntity().also {
            it.id = 2L
            it.agentId = "AI-2"
            it.agentStatus = status("development", 2L)
            it.disabled = true
        }

        every { terBankService.list() } returns emptyList()
        every { aiAgentRepository.findAll() } returns listOf(activeAgent, disabledAgent)
        every { statusRepository.findAllActive() } returns listOf(
            status("analysis", 1L),
            status("development", 2L),
            status("pilot", 3L),
            status("targetSolution", 4L),
        )
        every { agentStatusSlaRepository.findAllByAiAgentIdIn(any()) } returns
            listOf(sla("development", LocalDateTime.of(2026, 9, 1, 10, 0)))
        every { emailProperties.emailLinkProperties.linkToPortalShort } returns "https://pult.sber.ru/"

        // when
        val response = service.downloadRegistryExcel()

        // then
        verify(exactly = 1) { aiAgentRepository.findAll() }
        verify(exactly = 1) { agentStatusSlaRepository.findAllByAiAgentIdIn(listOf(1L, 2L)) }
        verify(exactly = 0) { agentStatusSlaRepository.findAllByAiAgentId(any()) }

        assertEquals(HttpStatus.OK, response.statusCode)
        assertEquals(InitiativeRegistryExportService.XLSX_MEDIA_TYPE, response.headers.contentType)
    }

    @Test
    fun `should set exact content disposition filename`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
        }

        every { terBankService.list() } returns emptyList()
        every { aiAgentRepository.findAll() } returns listOf(agent)
        every { statusRepository.findAllActive() } returns emptyList()
        every { agentStatusSlaRepository.findAllByAiAgentIdIn(any()) } returns emptyList()
        every { emailProperties.emailLinkProperties.linkToPortalShort } returns "https://pult.sber.ru/"

        // when
        val response = service.downloadRegistryExcel()

        // then
        val disposition = ContentDisposition.parse(response.headers.getFirst(HttpHeaders.CONTENT_DISPOSITION)!!)

        assertEquals(InitiativeRegistryExportService.FILE_NAME, disposition.filename)
    }

    @Test
    fun `should produce 28 columns in the required order`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
        }

        every { terBankService.list() } returns emptyList()
        every { aiAgentRepository.findAll() } returns listOf(agent)
        every { statusRepository.findAllActive() } returns emptyList()
        every { agentStatusSlaRepository.findAllByAiAgentIdIn(any()) } returns emptyList()
        every { emailProperties.emailLinkProperties.linkToPortalShort } returns "https://pult.sber.ru/"

        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val headerRow = workbook.getSheetAt(0).getRow(0)

            assertEquals(28, headerRow.physicalNumberOfCells)

            val expectedHeaders = listOf(
                "ID AI-агента",
                "Блок",
                "Трайб",
                "ТБ",
                "Наименование",
                "Описание",
                "Проблема, которую решает",
                "Текущий статус",
                "Ссылка JIRA",
                "CROSSGOAL",
                "GIGAUSAGE",
                "Тип инициативы",
                "Аудитория",
                "Программа AI-трансформации",
                "Фин. эффект, руб.",
                "Фин. эффект, ПШЕ",
                "Поверхности",
                "Архив",
                "Ссылка на инициативу в Пульт",
                "Email контакта",
                "Статус этапа — Концепция",
                "Дедлайн этапа — Концепция",
                "Статус этапа — PoC",
                "Дедлайн этапа — PoC",
                "Статус этапа — MVP",
                "Дедлайн этапа — MVP",
                "Статус этапа — Целевое решение",
                "Дедлайн этапа — Целевое решение",
            )

            val headers = (0 until 28).map { headerRow.getCell(it).stringCellValue }

            assertEquals(expectedHeaders, headers)
        }
    }

    @Test
    fun `should set active excel hyperlink for initiative url column`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 100L
            it.agentId = "AI-100"
            it.disabled = false
        }

        every { terBankService.list() } returns emptyList()
        every { aiAgentRepository.findAll() } returns listOf(agent)
        every { statusRepository.findAllActive() } returns emptyList()
        every { agentStatusSlaRepository.findAllByAiAgentIdIn(any()) } returns emptyList()
        every { emailProperties.emailLinkProperties.linkToPortalShort } returns "https://pult.sber.ru/"

        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val cell = workbook.getSheetAt(0).getRow(1).getCell(18)

            assertNotNull(cell.hyperlink)
            assertEquals("https://pult.sber.ru/ai/initiatives/100", cell.hyperlink.address)
            assertEquals(cell.hyperlink.address, cell.stringCellValue)
        }
    }

    @Test
    fun `should produce xlsx media type`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
        }

        every { terBankService.list() } returns emptyList()
        every { aiAgentRepository.findAll() } returns listOf(agent)
        every { statusRepository.findAllActive() } returns emptyList()
        every { agentStatusSlaRepository.findAllByAiAgentIdIn(any()) } returns emptyList()
        every { emailProperties.emailLinkProperties.linkToPortalShort } returns "https://pult.sber.ru/"

        // when
        val response = service.downloadRegistryExcel()

        // then
        assertEquals(
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            response.headers.contentType.toString(),
        )
    }

    @Test
    fun `blank portal base url throws`() {
        // given
        every { emailProperties.emailLinkProperties.linkToPortalShort } returns "   "

        // when
        val exception = assertThrows(AiInternalServerException::class.java) {
            service.downloadRegistryExcel()
        }

        // then
        assertEquals(
            "Property 'prm.email.link.link-to-portal-short' is not configured",
            exception.message,
        )

        verify(exactly = 0) { aiAgentRepository.findAll() }
    }

    @Test
    fun `portal base url without trailing slash should produce correct initiative url`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 100L
            it.agentId = "AI-100"
        }

        every { terBankService.list() } returns emptyList()
        every { aiAgentRepository.findAll() } returns listOf(agent)
        every { statusRepository.findAllActive() } returns emptyList()
        every { agentStatusSlaRepository.findAllByAiAgentIdIn(any()) } returns emptyList()
        every { emailProperties.emailLinkProperties.linkToPortalShort } returns "https://pult.sber.ru"

        // when
        val response = service.downloadRegistryExcel()

        // then
        XSSFWorkbook(response.body!!.inputStream).use { workbook ->
            val cell = workbook.getSheetAt(0).getRow(1).getCell(18)

            assertEquals("https://pult.sber.ru/ai/initiatives/100", cell.stringCellValue)
            assertEquals("https://pult.sber.ru/ai/initiatives/100", cell.hyperlink.address)
        }
    }

    @Test
    fun `valid portal base url produces export`() {
        // given
        val agent = AIAgentEntity().also {
            it.id = 1L
            it.agentId = "AI-1"
        }

        every { terBankService.list() } returns emptyList()
        every { aiAgentRepository.findAll() } returns listOf(agent)
        every { statusRepository.findAllActive() } returns emptyList()
        every { agentStatusSlaRepository.findAllByAiAgentIdIn(any()) } returns emptyList()
        every { emailProperties.emailLinkProperties.linkToPortalShort } returns "https://pult.sber.ru/"

        // when
        val response = service.downloadRegistryExcel()

        // then
        assertEquals(HttpStatus.OK, response.statusCode)
    }
}

```
