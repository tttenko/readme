```java

@ExtendWith(MockKExtension::class)
internal class EnablerServiceTest {

    @MockK
    private lateinit var enablerRepository: EnablerRepository

    @MockK
    private lateinit var messageProvider: MessageProvider

    @InjectMockKs
    private lateinit var service: EnablerService

    @Test
    fun `getEnablerReferences should return only enabled records by default`() {
        // given
        val entity = EnablerEntity().apply {
            id = 1L
            name = "Enabler 1"
            disabled = false
        }

        every { enablerRepository.findAllByDisabledIsFalse() } returns listOf(entity)

        // when
        val result = service.getEnablerReferences(includeDisabled = false)

        // then
        verify(exactly = 1) { enablerRepository.findAllByDisabledIsFalse() }
        verify(exactly = 0) { enablerRepository.findAll() }

        assertEquals(1, result.size)
        assertEquals(1L, result.first().id)
        assertEquals("Enabler 1", result.first().name)
        assertEquals(false, result.first().disabled)
    }

    @Test
    fun `getEnablerReferences should return enabled and disabled records when includeDisabled is true`() {
        // given
        val enabled = EnablerEntity().apply {
            id = 1L
            name = "Enabled enabler"
            disabled = false
        }

        val disabled = EnablerEntity().apply {
            id = 2L
            name = "Disabled enabler"
            this.disabled = true
        }

        every { enablerRepository.findAll() } returns listOf(enabled, disabled)

        // when
        val result = service.getEnablerReferences(includeDisabled = true)

        // then
        verify(exactly = 1) { enablerRepository.findAll() }
        verify(exactly = 0) { enablerRepository.findAllByDisabledIsFalse() }

        assertEquals(2, result.size)

        assertEquals(1L, result[0].id)
        assertEquals("Enabled enabler", result[0].name)
        assertEquals(false, result[0].disabled)

        assertEquals(2L, result[1].id)
        assertEquals("Disabled enabler", result[1].name)
        assertEquals(true, result[1].disabled)
    }

    @Test
    fun `getEnablerReferences should return empty list when records not found`() {
        // given
        every { enablerRepository.findAllByDisabledIsFalse() } returns emptyList()

        // when
        val result = service.getEnablerReferences(includeDisabled = false)

        // then
        verify(exactly = 1) { enablerRepository.findAllByDisabledIsFalse() }

        assertEquals(0, result.size)
    }

    @Test
    fun `createEnabler should save entity and return id`() {
        // given
        val request = CreateEnablerRequest(
            name = "Enabler 1"
        )

        val savedEntity = EnablerEntity().apply {
            id = 1L
            name = "Enabler 1"
            disabled = false
        }

        val entitySlot = slot<EnablerEntity>()

        every { enablerRepository.existsByNormalizedName(request.name) } returns false
        every { enablerRepository.save(capture(entitySlot)) } returns savedEntity

        // when
        val result = service.createEnabler(request)

        // then
        verify(exactly = 1) { enablerRepository.existsByNormalizedName(request.name) }
        verify(exactly = 1) { enablerRepository.save(any()) }

        assertEquals(CreateEnablerResponse(id = 1L), result)
        assertEquals("Enabler 1", entitySlot.captured.name)
        assertEquals(false, entitySlot.captured.disabled)
    }

    @Test
    fun `createEnabler should throw BAD_REQUEST when name already exists`() {
        // given
        val request = CreateEnablerRequest(
            name = "Enabler 1"
        )

        every { enablerRepository.existsByNormalizedName(request.name) } returns true
        every { messageProvider[ENABLER_NAME_ALREADY_EXISTS] } returns "Название {0} уже существует"

        // when
        val exception = assertThrows(AiBadRequestException::class.java) {
            service.createEnabler(request)
        }

        // then
        verify(exactly = 1) { enablerRepository.existsByNormalizedName(request.name) }
        verify(exactly = 1) { messageProvider[ENABLER_NAME_ALREADY_EXISTS] }
        verify(exactly = 0) { enablerRepository.save(any()) }

        assertEquals("Название Enabler 1 уже существует", exception.message)
    }

    @Test
    fun `updateEnabler should update existing entity and save it`() {
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

        val entitySlot = slot<EnablerEntity>()

        every { enablerRepository.findEnablerEntityById(id) } returns entity
        every { enablerRepository.existsByNormalizedNameAndIdNot(request.name, id) } returns false
        every { enablerRepository.save(capture(entitySlot)) } returns entity

        // when
        service.updateEnabler(id, request)

        // then
        verify(exactly = 1) { enablerRepository.findEnablerEntityById(id) }
        verify(exactly = 1) { enablerRepository.existsByNormalizedNameAndIdNot(request.name, id) }
        verify(exactly = 1) { enablerRepository.save(any()) }

        assertEquals(id, entitySlot.captured.id)
        assertEquals("Updated enabler", entitySlot.captured.name)
        assertEquals(true, entitySlot.captured.disabled)
    }

    @Test
    fun `updateEnabler should throw NOT_FOUND when entity does not exist`() {
        // given
        val id = 1L

        val request = UpdateEnablerRequest(
            name = "Updated enabler",
            disabled = false
        )

        every { enablerRepository.findEnablerEntityById(id) } returns null
        every { messageProvider[ENABLER_NOT_FOUND] } returns "Enabler с id {0} не найден"

        // when
        val exception = assertThrows(ResponseStatusException::class.java) {
            service.updateEnabler(id, request)
        }

        // then
        verify(exactly = 1) { enablerRepository.findEnablerEntityById(id) }
        verify(exactly = 1) { messageProvider[ENABLER_NOT_FOUND] }
        verify(exactly = 0) { enablerRepository.existsByNormalizedNameAndIdNot(any(), any()) }
        verify(exactly = 0) { enablerRepository.save(any()) }

        assertEquals(HttpStatus.NOT_FOUND, exception.statusCode)
        assertEquals("Enabler с id 1 не найден", exception.reason)
    }

    @Test
    fun `updateEnabler should throw BAD_REQUEST when name already exists`() {
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

        every { enablerRepository.findEnablerEntityById(id) } returns entity
        every { enablerRepository.existsByNormalizedNameAndIdNot(request.name, id) } returns true
        every { messageProvider[ENABLER_NAME_ALREADY_EXISTS] } returns "Название {0} уже существует"

        // when
        val exception = assertThrows(AiBadRequestException::class.java) {
            service.updateEnabler(id, request)
        }

        // then
        verify(exactly = 1) { enablerRepository.findEnablerEntityById(id) }
        verify(exactly = 1) { enablerRepository.existsByNormalizedNameAndIdNot(request.name, id) }
        verify(exactly = 1) { messageProvider[ENABLER_NAME_ALREADY_EXISTS] }
        verify(exactly = 0) { enablerRepository.save(any()) }

        assertEquals("Название Enabler 1 уже существует", exception.message)
    }

    @Test
    fun `toEnablerResponse should map entity to response`() {
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
    fun `toEnablerResponse should map null disabled as false`() {
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
