```java
@ExtendWith(MockKExtension::class)
class ContactUpdaterTest {

    @MockK
    lateinit var contactRepository: ContactRepository

    @MockK
    lateinit var contactValidator: ContactValidator

    @MockK
    lateinit var messageProvider: MessageProvider

    private lateinit var contactUpdater: ContactUpdater

    @BeforeEach
    fun setUp() {
        contactUpdater = ContactUpdater(
            contactRepository = contactRepository,
            contactValidator = contactValidator,
            messageProvider = messageProvider,
        )

        every {
            contactValidator.validate(
                contacts = any(),
                operationDetails = any(),
            )
        } just Runs

        every {
            messageProvider[CONTACT_NOT_FOUND]
        } returns "Указанный контакт не найден"

        every {
            messageProvider[CONTACT_NOT_DEFINED]
        } returns "Контакт не определён"
    }

    @Test
    fun `update should validate contacts before update`() {
        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = 1L,
                contact = null,
            )
        )

        val agent = AIAgentEntity().apply {
            id = 1L
            agentContact = mutableListOf()
        }

        contactUpdater.update(
            contacts = contacts,
            agent = agent,
            operationDetails = OPERATION_DETAILS,
        )

        verify(exactly = 1) {
            contactValidator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }
    }

    @Test
    fun `update should replace existing contacts with user contact`() {
        val oldContact = AgentContactEntity(
            agent = null,
            type = "old",
            contact = ContactEntity(
                fio = "Old Name",
                email = "old@example.com",
                invited = null,
            ),
            userId = null,
        )

        val agent = AIAgentEntity().apply {
            id = 1L
            agentContact = mutableListOf(oldContact)
        }

        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = 100L,
                contact = null,
            )
        )

        contactUpdater.update(
            contacts = contacts,
            agent = agent,
            operationDetails = OPERATION_DETAILS,
        )

        assertEquals(1, agent.agentContact.size)

        val actualContact = agent.agentContact.first()

        assertEquals(agent, actualContact.agent)
        assertEquals("owner", actualContact.type)
        assertEquals(100L, actualContact.userId)
        assertNull(actualContact.contact)
    }

    @Test
    fun `update should reuse existing contact by email and update fio`() {
        val existingContact = ContactEntity(
            fio = "Old Name",
            email = "test@example.com",
            invited = null,
        ).apply {
            id = 10L
        }

        val agent = AIAgentEntity().apply {
            id = 1L
            agentContact = mutableListOf()
        }

        val contacts = listOf(
            ContactInfoActionDto(
                type = "businessOwner",
                userId = null,
                contact = ContactDto(
                    id = null,
                    fio = "New Name",
                    email = "test@example.com",
                ),
            )
        )

        every {
            contactRepository.findFirstByEmail(
                email = "test@example.com",
            )
        } returns existingContact

        contactUpdater.update(
            contacts = contacts,
            agent = agent,
            operationDetails = OPERATION_DETAILS,
        )

        assertEquals(1, agent.agentContact.size)

        val actualAgentContact = agent.agentContact.first()

        assertEquals("businessOwner", actualAgentContact.type)
        assertNull(actualAgentContact.userId)
        assertEquals(existingContact, actualAgentContact.contact)
        assertEquals("New Name", existingContact.fio)
        assertEquals("test@example.com", existingContact.email)

        verify(exactly = 1) {
            contactRepository.findFirstByEmail(
                email = "test@example.com",
            )
        }

        verify(exactly = 0) {
            contactRepository.save(any<ContactEntity>())
        }
    }

    @Test
    fun `update should create new contact when contact with email not found`() {
        val agent = AIAgentEntity().apply {
            id = 1L
            agentContact = mutableListOf()
        }

        val contacts = listOf(
            ContactInfoActionDto(
                type = "businessOwner",
                userId = null,
                contact = ContactDto(
                    id = null,
                    fio = "Ivan Ivanov",
                    email = "ivan@example.com",
                ),
            )
        )

        every {
            contactRepository.findFirstByEmail(
                email = "ivan@example.com",
            )
        } returns null

        every {
            contactRepository.save(any<ContactEntity>())
        } answers {
            firstArg()
        }

        contactUpdater.update(
            contacts = contacts,
            agent = agent,
            operationDetails = OPERATION_DETAILS,
        )

        assertEquals(1, agent.agentContact.size)

        val actualAgentContact = agent.agentContact.first()
        val actualContact = requireNotNull(actualAgentContact.contact)

        assertEquals("businessOwner", actualAgentContact.type)
        assertNull(actualAgentContact.userId)
        assertEquals("Ivan Ivanov", actualContact.fio)
        assertEquals("ivan@example.com", actualContact.email)
        assertNull(actualContact.invited)

        verify(exactly = 1) {
            contactRepository.save(any<ContactEntity>())
        }
    }

    @Test
    fun `update should add contact relation by existing contact id`() {
        val existingContact = ContactEntity(
            fio = "Old Name",
            email = "existing@example.com",
            invited = null,
        ).apply {
            id = 10L
        }

        val agent = AIAgentEntity().apply {
            id = 1L
            agentContact = mutableListOf()
        }

        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = null,
                contact = ContactDto(
                    id = 10L,
                    fio = null,
                    email = null,
                ),
            )
        )

        every {
            contactRepository.findById(10L)
        } returns Optional.of(existingContact)

        contactUpdater.update(
            contacts = contacts,
            agent = agent,
            operationDetails = OPERATION_DETAILS,
        )

        assertEquals(1, agent.agentContact.size)

        val actualAgentContact = agent.agentContact.first()

        assertEquals("owner", actualAgentContact.type)
        assertNull(actualAgentContact.userId)
        assertEquals(existingContact, actualAgentContact.contact)
        assertEquals("Old Name", existingContact.fio)
        assertEquals("existing@example.com", existingContact.email)
    }

    @Test
    fun `update should update fio when contact found by id and fio is passed`() {
        val existingContact = ContactEntity(
            fio = "Old Name",
            email = "existing@example.com",
            invited = null,
        ).apply {
            id = 10L
        }

        val agent = AIAgentEntity().apply {
            id = 1L
            agentContact = mutableListOf()
        }

        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = null,
                contact = ContactDto(
                    id = 10L,
                    fio = "New Name",
                    email = "existing@example.com",
                ),
            )
        )

        every {
            contactRepository.findById(10L)
        } returns Optional.of(existingContact)

        contactUpdater.update(
            contacts = contacts,
            agent = agent,
            operationDetails = OPERATION_DETAILS,
        )

        assertEquals(1, agent.agentContact.size)
        assertEquals("New Name", existingContact.fio)
        assertEquals(existingContact, agent.agentContact.first().contact)
    }

    @Test
    fun `update should throw exception when contact by id not found`() {
        val agent = AIAgentEntity().apply {
            id = 1L
            agentContact = mutableListOf()
        }

        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = null,
                contact = ContactDto(
                    id = 999L,
                    fio = null,
                    email = null,
                ),
            )
        )

        every {
            contactRepository.findById(999L)
        } returns Optional.empty()

        val exception = assertThrows<AiBadRequestException> {
            contactUpdater.update(
                contacts = contacts,
                agent = agent,
                operationDetails = OPERATION_DETAILS,
            )
        }

        assertEquals(CONTACT_NOT_FOUND, exception.errorCode)
        assertEquals("Указанный контакт не найден", exception.message)
    }

    @Test
    fun `update should throw exception when contact email does not match existing contact email`() {
        val existingContact = ContactEntity(
            fio = "Old Name",
            email = "existing@example.com",
            invited = null,
        ).apply {
            id = 10L
        }

        val agent = AIAgentEntity().apply {
            id = 1L
            agentContact = mutableListOf()
        }

        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = null,
                contact = ContactDto(
                    id = 10L,
                    fio = null,
                    email = "wrong@example.com",
                ),
            )
        )

        every {
            contactRepository.findById(10L)
        } returns Optional.of(existingContact)

        val exception = assertThrows<AiBadRequestException> {
            contactUpdater.update(
                contacts = contacts,
                agent = agent,
                operationDetails = OPERATION_DETAILS,
            )
        }

        assertEquals(CONTACT_NOT_DEFINED, exception.errorCode)
        assertEquals("Контакт не определён", exception.message)
    }

    @Test
    fun `update should not clear contacts when validator throws exception`() {
        val oldAgentContact = AgentContactEntity(
            agent = null,
            type = "old",
            contact = ContactEntity(
                fio = "Old Name",
                email = "old@example.com",
                invited = null,
            ),
            userId = null,
        )

        val agent = AIAgentEntity().apply {
            id = 1L
            agentContact = mutableListOf(oldAgentContact)
        }

        val contacts = listOf(
            ContactInfoActionDto(
                type = null,
                userId = null,
                contact = null,
            )
        )

        every {
            contactValidator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        } throws AiBadRequestException(
            errorCode = NoMandatoryField,
            message = "Отсутствуют обязательные поля",
            operationDetails = OPERATION_DETAILS,
        )

        assertThrows<AiBadRequestException> {
            contactUpdater.update(
                contacts = contacts,
                agent = agent,
                operationDetails = OPERATION_DETAILS,
            )
        }

        assertEquals(1, agent.agentContact.size)
        assertEquals(oldAgentContact, agent.agentContact.first())
    }

    private companion object {
        const val OPERATION_DETAILS = "Patch agent contacts"
    }
}

  
```
