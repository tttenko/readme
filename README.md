```java

import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.ExtendWith
import org.mockito.InjectMocks
import org.mockito.Mock
import org.mockito.Mockito.verify
import org.mockito.junit.jupiter.MockitoExtension
import org.mockito.kotlin.any
import org.mockito.kotlin.argumentCaptor
import org.mockito.kotlin.never
import org.mockito.kotlin.whenever
import org.springframework.http.HttpStatus
import org.springframework.web.server.ResponseStatusException
import ru.sber.prm.dto.references.enabler.CreateEnablerRequest
import ru.sber.prm.dto.references.enabler.CreateEnablerResponse
import ru.sber.prm.dto.references.enabler.EnablerResponse
import ru.sber.prm.dto.references.enabler.UpdateEnablerRequest
import ru.sber.prm.entity.references.EnablerEntity
import ru.sber.prm.exception.AiBadRequestException
import ru.sber.prm.repository.references.EnablerRepository
import ru.sber.prm.service.references.EnablerService
import ru.sber.prm.service.references.toEnablerResponse
import ru.sber.prm.service.MessageProvider
import ru.sber.prm.util.MessageCode.ENABLER_NAME_ALREADY_EXISTS
import ru.sber.prm.util.MessageCode.ENABLER_NOT_FOUND

@ExtendWith(MockitoExtension::class)
internal class EnablerServiceTest {

    @Mock
    private lateinit var enablerRepository: EnablerRepository

    @Mock
    private lateinit var messageProvider: MessageProvider

    @InjectMocks
    private lateinit var service: EnablerService

    @Test
    fun `should get only active enabler references when include disabled is false`() {
        // given
        val entity = EnablerEntity().apply {
            id = 1L
            name = "Enabler 1"
            disabled = false
        }

        whenever(enablerRepository.findAllByDisabledIsFalse())
            .thenReturn(listOf(entity))

        // when
        val result = service.getEnablerReferences(includeDisabled = false)

        // then
        assertEquals(1, result.size)
        assertEquals(1L, result[0].id)
        assertEquals("Enabler 1", result[0].name)
        assertEquals(false, result[0].disabled)

        verify(enablerRepository).findAllByDisabledIsFalse()
        verify(enablerRepository, never()).findAll()
    }

    @Test
    fun `should get all enabler references when include disabled is true`() {
        // given
        val activeEntity = EnablerEntity().apply {
            id = 1L
            name = "Active enabler"
            disabled = false
        }

        val disabledEntity = EnablerEntity().apply {
            id = 2L
            name = "Disabled enabler"
            disabled = true
        }

        whenever(enablerRepository.findAll())
            .thenReturn(listOf(activeEntity, disabledEntity))

        // when
        val result = service.getEnablerReferences(includeDisabled = true)

        // then
        assertEquals(2, result.size)

        assertEquals(1L, result[0].id)
        assertEquals("Active enabler", result[0].name)
        assertEquals(false, result[0].disabled)

        assertEquals(2L, result[1].id)
        assertEquals("Disabled enabler", result[1].name)
        assertEquals(true, result[1].disabled)

        verify(enablerRepository).findAll()
        verify(enablerRepository, never()).findAllByDisabledIsFalse()
    }

    @Test
    fun `should return empty list when enablers not found`() {
        // given
        whenever(enablerRepository.findAllByDisabledIsFalse())
            .thenReturn(emptyList())

        // when
        val result = service.getEnablerReferences(includeDisabled = false)

        // then
        assertEquals(0, result.size)

        verify(enablerRepository).findAllByDisabledIsFalse()
    }

    @Test
    fun `should create enabler with success`() {
        // given
        val request = CreateEnablerRequest(
            name = "Enabler 1"
        )

        val savedEntity = EnablerEntity().apply {
            id = 1L
            name = "Enabler 1"
            disabled = false
        }

        whenever(enablerRepository.existsByNormalizedName(request.name))
            .thenReturn(false)

        whenever(enablerRepository.save(any<EnablerEntity>()))
            .thenReturn(savedEntity)

        // when
        val result = service.createEnabler(request)

        // then
        assertEquals(CreateEnablerResponse(id = 1L), result)

        val entityCaptor = argumentCaptor<EnablerEntity>()
        verify(enablerRepository).save(entityCaptor.capture())

        assertEquals("Enabler 1", entityCaptor.firstValue.name)
        assertEquals(false, entityCaptor.firstValue.disabled)
    }

    @Test
    fun `should throw bad request when create enabler name already exists`() {
        // given
        val request = CreateEnablerRequest(
            name = "Enabler 1"
        )

        whenever(enablerRepository.existsByNormalizedName(request.name))
            .thenReturn(true)

        whenever(messageProvider[ENABLER_NAME_ALREADY_EXISTS])
            .thenReturn("Название {0} уже существует")

        // when
        val exception = assertThrows(AiBadRequestException::class.java) {
            service.createEnabler(request)
        }

        // then
        assertEquals("Название Enabler 1 уже существует", exception.message)

        verify(enablerRepository).existsByNormalizedName(request.name)
        verify(enablerRepository, never()).save(any<EnablerEntity>())
    }

    @Test
    fun `should update enabler with success`() {
        // given
        val id = 1L

        val request = UpdateEnablerRequest(
            name = "Updated enabler",
            disabled = true
        )

        val entity = EnablerEntity().apply {
            this.id = id
            name = "Old enabler"
            disabled = false
        }

        whenever(enablerRepository.findEnablerEntityById(id))
            .thenReturn(entity)

        whenever(enablerRepository.existsByNormalizedNameAndIdNot(request.name, id))
            .thenReturn(false)

        whenever(enablerRepository.save(any<EnablerEntity>()))
            .thenAnswer { invocation -> invocation.getArgument(0) }

        // when
        service.updateEnabler(id, request)

        // then
        val entityCaptor = argumentCaptor<EnablerEntity>()
        verify(enablerRepository).save(entityCaptor.capture())

        assertEquals(id, entityCaptor.firstValue.id)
        assertEquals("Updated enabler", entityCaptor.firstValue.name)
        assertEquals(true, entityCaptor.firstValue.disabled)
    }

    @Test
    fun `should throw not found when update enabler entity not found`() {
        // given
        val id = 1L

        val request = UpdateEnablerRequest(
            name = "Updated enabler",
            disabled = false
        )

        whenever(enablerRepository.findEnablerEntityById(id))
            .thenReturn(null)

        whenever(messageProvider[ENABLER_NOT_FOUND])
            .thenReturn("Enabler с id {0} не найден")

        // when
        val exception = assertThrows(ResponseStatusException::class.java) {
            service.updateEnabler(id, request)
        }

        // then
        assertEquals(HttpStatus.NOT_FOUND, exception.statusCode)
        assertEquals("Enabler с id 1 не найден", exception.reason)

        verify(enablerRepository).findEnablerEntityById(id)
        verify(enablerRepository, never()).save(any<EnablerEntity>())
    }

    @Test
    fun `should throw bad request when update enabler name already exists`() {
        // given
        val id = 1L

        val request = UpdateEnablerRequest(
            name = "Enabler 1",
            disabled = true
        )

        val entity = EnablerEntity().apply {
            this.id = id
            name = "Old enabler"
            disabled = false
        }

        whenever(enablerRepository.findEnablerEntityById(id))
            .thenReturn(entity)

        whenever(enablerRepository.existsByNormalizedNameAndIdNot(request.name, id))
            .thenReturn(true)

        whenever(messageProvider[ENABLER_NAME_ALREADY_EXISTS])
            .thenReturn("Название {0} уже существует")

        // when
        val exception = assertThrows(AiBadRequestException::class.java) {
            service.updateEnabler(id, request)
        }

        // then
        assertEquals("Название Enabler 1 уже существует", exception.message)

        verify(enablerRepository).findEnablerEntityById(id)
        verify(enablerRepository).existsByNormalizedNameAndIdNot(request.name, id)
        verify(enablerRepository, never()).save(any<EnablerEntity>())
    }

    @Test
    fun `should map enabler entity to enabler response`() {
        // given
        val entity = EnablerEntity().apply {
            id = 1L
            name = "Enabler 1"
            disabled = true
        }

        // when
        val result = entity.toEnablerResponse()

        // then
        assertEquals(
            EnablerResponse(
                id = 1L,
                name = "Enabler 1",
                disabled = true
            ),
            result
        )
    }

    @Test
    fun `should map null disabled as false`() {
        // given
        val entity = EnablerEntity().apply {
            id = 1L
            name = "Enabler 1"
            disabled = null
        }

        // when
        val result = entity.toEnablerResponse()

        // then
        assertEquals(1L, result.id)
        assertEquals("Enabler 1", result.name)
        assertEquals(false, result.disabled)
    }
}
```
