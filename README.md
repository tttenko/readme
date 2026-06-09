```java

@Component
open class EnablerUpdater(
    private val enablerRepository: EnablerRepository,
    private val messageProvider: MessageProvider,
) {

    open fun update(
        agent: AIAgentEntity,
        rawEnablersValue: Any?,
    ) {
        val requestedEnablerIds = parseEnablerIds(
            rawEnablersValue = rawEnablersValue,
        )

        /*
         * null или пустой список означает удаление
         * всех связей агента с энейблерами.
         */
        if (requestedEnablerIds.isEmpty()) {
            agent.enablers.clear()
            return
        }

        val requestedEnablers =
            enablerRepository.findAllById(
                requestedEnablerIds
            )

        validateRequestedEnablersExist(
            requestedEnablerIds = requestedEnablerIds,
            requestedEnablers = requestedEnablers,
        )

        /*
         * AIAgentEntity является владельцем ManyToMany-связи,
         * поэтому Hibernate самостоятельно синхронизирует
         * таблицу agent_enabler при сохранении агента.
         */
        agent.enablers.clear()
        agent.enablers.addAll(requestedEnablers)
    }

    private fun parseEnablerIds(
        rawEnablersValue: Any?,
    ): Set<Long> {
        if (rawEnablersValue == null) {
            return emptySet()
        }

        val rawEnablers =
            rawEnablersValue as? List<*>
                ?: throwWrongEnablersValue()

        return rawEnablers
            .map { rawEnablerId ->
                val enablerId =
                    (rawEnablerId as? Number)?.toLong()
                        ?: throwWrongEnablersValue()

                if (enablerId <= 0) {
                    throwWrongEnablersValue()
                }

                enablerId
            }
            .toSet()
    }

    private fun validateRequestedEnablersExist(
        requestedEnablerIds: Set<Long>,
        requestedEnablers: List<EnablerEntity>,
    ) {
        val existingEnablerIds =
            requestedEnablers
                .mapNotNull { enabler ->
                    enabler.id
                }
                .toSet()

        val missingEnablerIds =
            requestedEnablerIds - existingEnablerIds

        if (missingEnablerIds.isNotEmpty()) {
            throw AiBadRequestException(
                errorCode = ENABLER_WITH_ID_NOT_FOUND,
                message = MessageFormat.format(
                    messageProvider[ENABLER_WITH_ID_NOT_FOUND],
                    missingEnablerIds.joinToString(),
                ),
            )
        }
    }

    private fun throwWrongEnablersValue(): Nothing {
        throw AiBadRequestException(
            errorCode = WRONG_ENABLERS_VALUE,
            message = messageProvider[WRONG_ENABLERS_VALUE],
        )
    }
}

/**
 * Обновляет связи AI-агента с энейблерами по переданному списку идентификаторов.
 *
 * Проверяет корректность значений и существование энейблеров в справочнике.
 * `null` или пустой список удаляет все текущие связи агента с энейблерами.
 */
@Component
open class EnablerUpdater(
    private val enablerRepository: EnablerRepository,
    private val messageProvider: MessageProvider,
) {

    /**
     * Заменяет текущий набор энейблеров агента переданным набором.
     */
    open fun update(
        agent: AIAgentEntity,
        rawEnablersValue: Any?,
    ) {
        // ...
    }

    /**
     * Преобразует входное значение в уникальный набор идентификаторов энейблеров.
     */
    private fun parseEnablerIds(
        rawEnablersValue: Any?,
    ): Set<Long> {
        // ...
    }

    /**
     * Проверяет существование всех запрошенных энейблеров в справочнике.
     */
    private fun validateRequestedEnablersExist(
        requestedEnablerIds: Set<Long>,
        requestedEnablers: List<EnablerEntity>,
    ) {
        // ...
    }

    /**
     * Выбрасывает ошибку некорректного значения поля `enablers`.
     */
    private fun throwWrongEnablersValue(): Nothing {
        // ...
    }
}

```
