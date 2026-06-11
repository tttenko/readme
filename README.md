```java
@ExtendWith(MockKExtension::class)
class ContactValidatorTest {

    @MockK
    lateinit var messageProvider: MessageProvider

    private lateinit var validator: ContactValidator

    @BeforeEach
    fun setUp() {
        validator = ContactValidator(
            messageProvider = messageProvider
        )

        every {
            messageProvider[NoMandatoryField]
        } returns "Отсутствуют обязательные поля"

        every {
            messageProvider[DUPLICATE_CONTACT_TYPE]
        } returns "Нельзя создать два контакта с одним типом"
    }

    @Test
    fun `validate should pass when contact has type and userId`() {
        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = 1L,
                contact = null,
            )
        )

        assertDoesNotThrow {
            validator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }
    }

    @Test
    fun `validate should pass when contact has type and contact with email`() {
        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = null,
                contact = ContactDto(
                    id = null,
                    fio = "Ivan Ivanov",
                    email = "test@example.com",
                ),
            )
        )

        assertDoesNotThrow {
            validator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }
    }

    @Test
    fun `validate should pass when contact has type and contact with id`() {
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

        assertDoesNotThrow {
            validator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }
    }

    @Test
    fun `validate should pass when contacts list is empty`() {
        assertDoesNotThrow {
            validator.validate(
                contacts = emptyList(),
                operationDetails = OPERATION_DETAILS,
            )
        }
    }

    @Test
    fun `validate should throw exception when type is null`() {
        val contacts = listOf(
            ContactInfoActionDto(
                type = null,
                userId = 1L,
                contact = null,
            )
        )

        val exception = assertThrows<AiBadRequestException> {
            validator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }

        assertEquals(NoMandatoryField, exception.errorCode)
        assertEquals("Отсутствуют обязательные поля", exception.message)
    }

    @Test
    fun `validate should throw exception when type is blank`() {
        val contacts = listOf(
            ContactInfoActionDto(
                type = "   ",
                userId = 1L,
                contact = null,
            )
        )

        val exception = assertThrows<AiBadRequestException> {
            validator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }

        assertEquals(NoMandatoryField, exception.errorCode)
        assertEquals("Отсутствуют обязательные поля", exception.message)
    }

    @Test
    fun `validate should throw exception when userId and contact are null`() {
        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = null,
                contact = null,
            )
        )

        val exception = assertThrows<AiBadRequestException> {
            validator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }

        assertEquals(NoMandatoryField, exception.errorCode)
        assertEquals("Отсутствуют обязательные поля", exception.message)
    }

    @Test
    fun `validate should throw exception when userId and contact are both filled`() {
        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = 1L,
                contact = ContactDto(
                    id = null,
                    fio = "Ivan Ivanov",
                    email = "test@example.com",
                ),
            )
        )

        val exception = assertThrows<AiBadRequestException> {
            validator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }

        assertEquals(NoMandatoryField, exception.errorCode)
        assertEquals("Отсутствуют обязательные поля", exception.message)
    }

    @Test
    fun `validate should throw exception when contact has no id and no email`() {
        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = null,
                contact = ContactDto(
                    id = null,
                    fio = "Ivan Ivanov",
                    email = null,
                ),
            )
        )

        val exception = assertThrows<AiBadRequestException> {
            validator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }

        assertEquals(NoMandatoryField, exception.errorCode)
        assertEquals("Отсутствуют обязательные поля", exception.message)
    }

    @Test
    fun `validate should throw exception when contact has no id and blank email`() {
        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = null,
                contact = ContactDto(
                    id = null,
                    fio = "Ivan Ivanov",
                    email = "   ",
                ),
            )
        )

        val exception = assertThrows<AiBadRequestException> {
            validator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }

        assertEquals(NoMandatoryField, exception.errorCode)
        assertEquals("Отсутствуют обязательные поля", exception.message)
    }

    @Test
    fun `validate should throw exception when contacts have duplicate types`() {
        val contacts = listOf(
            ContactInfoActionDto(
                type = "owner",
                userId = 1L,
                contact = null,
            ),
            ContactInfoActionDto(
                type = "owner",
                userId = 2L,
                contact = null,
            )
        )

        val exception = assertThrows<AiBadRequestException> {
            validator.validate(
                contacts = contacts,
                operationDetails = OPERATION_DETAILS,
            )
        }

        assertEquals(DUPLICATE_CONTACT_TYPE, exception.errorCode)
        assertEquals(
            "Нельзя создать два контакта с одним типом",
            exception.message,
        )
    }

    private companion object {
        const val OPERATION_DETAILS = "Patch agent contacts"
    }
}


  
```
