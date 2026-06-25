```java
@ExtendWith(MockKExtension::class)
class InitiativeAgentTypesPermissionValidatorTest {

    @MockK
    private lateinit var userInfoProvider: UserInfoProvider

    @MockK
    private lateinit var statusRepository: StatusRepository

    private lateinit var validator: InitiativeAgentTypesPermissionValidator

    @BeforeEach
    fun setUp() {
        validator = InitiativeAgentTypesPermissionValidator(
            userInfoProvider = userInfoProvider,
            statusRepository = statusRepository,
        )
    }

    @Test
    fun `validate should not throw when initiative status is before MVP`() {
        // Given
        val initiative =
            createInitiative(
                statusOrdering = 30L
            )

        every {
            statusRepository.findFirstByCode(code = "pilot")
        } returns createStatus(
            code = "pilot",
            ordering = 40L,
        )

        // When & Then
        assertDoesNotThrow {
            validator.validate(initiative = initiative)
        }

        verify(exactly = 0) {
            userInfoProvider.currentUser()
        }
    }

    @Test
    fun `validate should not throw when initiative status is MVP and user has transformation office role`() {
        // Given
        val initiative =
            createInitiative(
                statusOrdering = 40L
            )

        every {
            statusRepository.findFirstByCode(code = "pilot")
        } returns createStatus(
            code = "pilot",
            ordering = 40L,
        )

        every {
            userInfoProvider.currentUser().roles
        } returns setOf("TRANSFORMATION_OFFICE")

        // When & Then
        assertDoesNotThrow {
            validator.validate(initiative = initiative)
        }

        verify(exactly = 1) {
            userInfoProvider.currentUser()
        }
    }

    @Test
    fun `validate should not throw when initiative status is after MVP and user has transformation office role`() {
        // Given
        val initiative =
            createInitiative(
                statusOrdering = 50L
            )

        every {
            statusRepository.findFirstByCode(code = "pilot")
        } returns createStatus(
            code = "pilot",
            ordering = 40L,
        )

        every {
            userInfoProvider.currentUser().roles
        } returns setOf("TRANSFORMATION_OFFICE")

        // When & Then
        assertDoesNotThrow {
            validator.validate(initiative = initiative)
        }

        verify(exactly = 1) {
            userInfoProvider.currentUser()
        }
    }

    @Test
    fun `validate should throw forbidden when initiative status is MVP and user has no transformation office role`() {
        // Given
        val initiative =
            createInitiative(
                statusOrdering = 40L
            )

        every {
            statusRepository.findFirstByCode(code = "pilot")
        } returns createStatus(
            code = "pilot",
            ordering = 40L,
        )

        every {
            userInfoProvider.currentUser().roles
        } returns setOf("PROJECT_OFFICE")

        // When & Then
        assertThrows<AiForbiddenException> {
            validator.validate(initiative = initiative)
        }

        verify(exactly = 1) {
            userInfoProvider.currentUser()
        }
    }

    @Test
    fun `validate should throw forbidden when initiative status is after MVP and user has no transformation office role`() {
        // Given
        val initiative =
            createInitiative(
                statusOrdering = 60L
            )

        every {
            statusRepository.findFirstByCode(code = "pilot")
        } returns createStatus(
            code = "pilot",
            ordering = 40L,
        )

        every {
            userInfoProvider.currentUser().roles
        } returns setOf("CMS_ADMIN")

        // When & Then
        assertThrows<AiForbiddenException> {
            validator.validate(initiative = initiative)
        }

        verify(exactly = 1) {
            userInfoProvider.currentUser()
        }
    }

    @Test
    fun `validate should not throw when initiative has no status`() {
        // Given
        val initiative =
            AIAgentEntity().apply {
                id = 1L
                agentStatus = null
            }

        // When & Then
        assertDoesNotThrow {
            validator.validate(initiative = initiative)
        }

        verify(exactly = 0) {
            statusRepository.findFirstByCode(any())
        }

        verify(exactly = 0) {
            userInfoProvider.currentUser()
        }
    }

    @Test
    fun `validate should not throw when mvp status not found`() {
        // Given
        val initiative =
            createInitiative(
                statusOrdering = 50L
            )

        every {
            statusRepository.findFirstByCode(code = "pilot")
        } returns null

        // When & Then
        assertDoesNotThrow {
            validator.validate(initiative = initiative)
        }

        verify(exactly = 0) {
            userInfoProvider.currentUser()
        }
    }

    private fun createInitiative(
        statusOrdering: Long,
    ): AIAgentEntity {
        return AIAgentEntity().apply {
            id = 1L
            agentStatus = createStatus(
                code = "development",
                ordering = statusOrdering,
            )
        }
    }

    private fun createStatus(
        code: String,
        ordering: Long,
    ): StatusEntity {
        return StatusEntity().apply {
            id = ordering
            name = code
            this.code = code
            this.ordering = ordering
            disabled = false
        }
    }
}
```
