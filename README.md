```java

/**
 * Обновляет связанные с AI-агентом контактные лица.
 *
 * Для контактов с {@code userId} создаёт связь с пользователем.
 * Для обычных контактов находит или создаёт запись в таблице контактов,
 * после чего создаёт связь контакта с агентом.
 *
 * Перед обновлением выполняет валидацию входных данных и полностью
 * заменяет текущий список контактов агента.
 */
@Component
open class ContactUpdater(
    private val contactRepository: ContactRepository,
    private val contactValidator: ContactValidator,
    private val messageProvider: MessageProvider,
) {

    /**
     * Валидирует переданные контакты и заменяет ими текущие контакты агента.
     *
     * @param contacts новый список контактов агента
     * @param agent обновляемый AI-агент
     * @param operationDetails описание выполняемой операции для формирования ошибок
     */
    open fun update(
        contacts: List<ContactInfoActionDto>,
        agent: AIAgentEntity,
        operationDetails: String,
    ) {
        contactValidator.validate(
            contacts = contacts,
            operationDetails = operationDetails,
        )

        agent.agentContact.clear()

        contacts.forEach { contact ->

            val contactType =
                requireNotNull(contact.type)

            val userId = contact.userId

            if (userId != null) {
                addUserContact(
                    agent = agent,
                    contactType = contactType,
                    userId = userId,
                )

                return@forEach
            }

            val contactDto =
                requireNotNull(contact.contact)

            if (contactDto.id == null) {
                addContactWithoutId(
                    agent = agent,
                    contactType = contactType,
                    contactDto = contactDto,
                )
            } else {
                addContactById(
                    agent = agent,
                    contactType = contactType,
                    contactDto = contactDto,
                    operationDetails = operationDetails,
                )
            }
        }
    }

    /**
     * Создаёт связь AI-агента с пользователем, указанным в качестве контакта.
     *
     * @param agent AI-агент
     * @param contactType тип контактного лица
     * @param userId идентификатор пользователя
     */
    private fun addUserContact(
        agent: AIAgentEntity,
        contactType: String,
        userId: Long,
    ) {
        agent.agentContact.add(
            AgentContactEntity(
                agent = agent,
                type = contactType,
                contact = null,
                userId = userId,
            )
        )
    }

    /**
     * Находит контакт по email или создаёт новую запись, если контакт не найден,
     * после чего связывает его с AI-агентом.
     *
     * Для существующего контакта обновляет ФИО значением из запроса.
     *
     * @param agent AI-агент
     * @param contactType тип контактного лица
     * @param contactDto данные контакта без идентификатора
     */
    private fun addContactWithoutId(
        agent: AIAgentEntity,
        contactType: String,
        contactDto: ContactDto,
    ) {
        val email =
            requireNotNull(contactDto.email)

        val contactEntity =
            contactRepository
                .findFirstByEmail(email)
                ?.apply {
                    fio = contactDto.fio
                }
                ?: contactRepository.save(
                    entity = ContactEntity(
                        fio = contactDto.fio,
                        email = email,
                        invited = null,
                    )
                )

        addContactRelation(
            agent = agent,
            contactType = contactType,
            contactEntity = contactEntity,
        )
    }

    /**
     * Находит контакт по идентификатору, проверяет соответствие email
     * и создаёт связь найденного контакта с AI-агентом.
     *
     * Если контакт не найден или переданный email не совпадает с сохранённым,
     * выбрасывает ошибку некорректного запроса.
     *
     * @param agent AI-агент
     * @param contactType тип контактного лица
     * @param contactDto данные существующего контакта
     * @param operationDetails описание выполняемой операции для формирования ошибок
     */
    private fun addContactById(
        agent: AIAgentEntity,
        contactType: String,
        contactDto: ContactDto,
        operationDetails: String,
    ) {
        val contactId =
            requireNotNull(contactDto.id)

        val contactEntity =
            contactRepository.findByIdOrNull(
                id = contactId,
            )
                ?: throw AiBadRequestException(
                    errorCode =
                        Metadata.ErrorMessages.CONTACT_NOT_FOUND,
                    message = messageProvider[
                        Metadata.ErrorMessages.CONTACT_NOT_FOUND
                    ],
                    operationDetails = operationDetails,
                )

        if (
            contactDto.email != null &&
            contactEntity.email != contactDto.email
        ) {
            throw AiBadRequestException(
                errorCode =
                    Metadata.ErrorMessages.CONTACT_NOT_DEFINED,
                message = messageProvider[
                    Metadata.ErrorMessages.CONTACT_NOT_DEFINED
                ],
                operationDetails = operationDetails,
            )
        }

        contactDto.fio?.let { requestedFio ->
            contactEntity.fio = requestedFio
        }

        addContactRelation(
            agent = agent,
            contactType = contactType,
            contactEntity = contactEntity,
        )
    }

    /**
     * Создаёт связь между AI-агентом и существующим контактом.
     *
     * @param agent AI-агент
     * @param contactType тип контактного лица
     * @param contactEntity связываемый контакт
     */
    private fun addContactRelation(
        agent: AIAgentEntity,
        contactType: String,
        contactEntity: ContactEntity,
    ) {
        agent.agentContact.add(
            AgentContactEntity(
                agent = agent,
                type = contactType,
                contact = contactEntity,
                userId = null,
            )
        )
    }
}
ContactValidator
/**
 * Проверяет корректность контактных лиц, переданных для обновления AI-агента.
 *
 * Валидирует наличие обязательных полей, корректность способа указания
 * контакта и отсутствие нескольких контактов с одинаковым типом.
 */
@Component
open class ContactValidator(
    private val messageProvider: MessageProvider,
) {

    /**
     * Выполняет все проверки переданного списка контактов.
     *
     * @param contacts проверяемые контакты
     * @param operationDetails описание выполняемой операции для формирования ошибок
     */
    open fun validate(
        contacts: List<ContactInfoActionDto>,
        operationDetails: String,
    ) {
        validateRequiredFields(
            contacts = contacts,
            operationDetails = operationDetails,
        )

        validateDuplicateTypes(
            contacts = contacts,
            operationDetails = operationDetails,
        )
    }

    /**
     * Проверяет заполнение обязательных полей каждого контакта.
     *
     * Для контакта должен быть указан тип и ровно один источник:
     * идентификатор пользователя либо блок с данными контакта.
     * Для нового контакта без идентификатора также обязателен email.
     *
     * @param contacts проверяемые контакты
     * @param operationDetails описание выполняемой операции для формирования ошибок
     */
    private fun validateRequiredFields(
        contacts: List<ContactInfoActionDto>,
        operationDetails: String,
    ) {
        contacts.forEach { contact ->

            val hasType =
                !contact.type.isNullOrBlank()

            val hasUserId =
                contact.userId != null

            val hasContact =
                contact.contact != null

            if (
                !hasType ||
                hasUserId == hasContact
            ) {
                throwRequiredFieldsMissing(
                    operationDetails = operationDetails,
                )
            }

            if (
                hasContact &&
                contact.contact?.id == null &&
                contact.contact?.email.isNullOrBlank()
            ) {
                throwRequiredFieldsMissing(
                    operationDetails = operationDetails,
                )
            }
        }
    }

    /**
     * Проверяет отсутствие нескольких контактов с одинаковым типом.
     *
     * @param contacts проверяемые контакты
     * @param operationDetails описание выполняемой операции для формирования ошибок
     */
    private fun validateDuplicateTypes(
        contacts: List<ContactInfoActionDto>,
        operationDetails: String,
    ) {
        val duplicatedTypes = contacts
            .mapNotNull { contact ->
                contact.type
            }
            .groupingBy { type ->
                type
            }
            .eachCount()
            .filterValues { count ->
                count > 1
            }

        if (duplicatedTypes.isNotEmpty()) {
            throw AiBadRequestException(
                errorCode = DUPLICATE_CONTACT_TYPE,
                message = messageProvider[DUPLICATE_CONTACT_TYPE],
                operationDetails = operationDetails,
            )
        }
    }

    /**
     * Выбрасывает ошибку отсутствия обязательных полей контакта.
     *
     * @param operationDetails описание выполняемой операции
     */
    private fun throwRequiredFieldsMissing(
        operationDetails: String,
    ): Nothing {
        throw AiBadRequestException(
            errorCode = REQUIRED_CONTACT_FIELDS_MISSING,
            message = messageProvider[
                REQUIRED_CONTACT_FIELDS_MISSING
            ],
            operationDetails = operationDetails,
        )
    }
}

```
