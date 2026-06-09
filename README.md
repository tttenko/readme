```java

@Component
open class ContactValidator(
    private val messageProvider: MessageProvider,
) {

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

            /*
             * Обязателен type и должен быть указан
             * ровно один вариант:
             * либо userId, либо contact.
             */
            if (
                !hasType ||
                hasUserId == hasContact
            ) {
                throwRequiredFieldsMissing(
                    operationDetails = operationDetails,
                )
            }

            /*
             * Для нового контакта, у которого нет ID,
             * email обязателен: по нему выполняется
             * поиск или создание ContactEntity.
             */
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

@Component
open class ContactUpdater(
    private val contactRepository: ContactRepository,
    private val contactValidator: ContactValidator,
    private val messageProvider: MessageProvider,
) {

    open fun update(
        contacts: List<ContactInfoActionDto>,
        agent: AIAgentEntity,
        operationDetails: String,
    ) {
        /*
         * Сначала полностью валидируем входные данные.
         * Состояние агента до успешной валидации
         * не изменяем.
         */
        contactValidator.validate(
            contacts = contacts,
            operationDetails = operationDetails,
        )

        /*
         * По документации связи агента с контактами
         * обновляются полной заменой.
         */
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

    private fun addContactWithoutId(
        agent: AIAgentEntity,
        contactType: String,
        contactDto: ContactDto,
    ) {
        val email =
            requireNotNull(contactDto.email)

        /*
         * Если контакт с таким email уже есть,
         * обновляем его ФИО.
         *
         * Если контакта нет, создаём новую запись.
         */
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

        /*
         * По документации email сверяется только тогда,
         * когда он присутствует в запросе.
         */
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

        /*
         * Сохраняем существующее поведение:
         * если ФИО пришло в запросе, обновляем его.
         */
        contactDto.fio?.let { requestedFio ->
            contactEntity.fio = requestedFio
        }

        addContactRelation(
            agent = agent,
            contactType = contactType,
            contactEntity = contactEntity,
        )
    }

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

```
