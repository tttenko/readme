```java

/**
 * Синхронизирует связанные с AI-агентом JIRA-инициативы проекта GIGAUSAGE.
 *
 * Компонент валидирует входные значения, создаёт и обновляет актуальные
 * записи `jira_issue`, а также удаляет записи, отсутствующие в запросе.
 */
@Component
open class GigausageIssueUpdater(
    private val jiraIssueRepository: JiraIssueRepository,
    private val messageProvider: MessageProvider,
) {

    /**
     * Обновляет GIGAUSAGE-задачи агента по данным PATCH-запроса.
     *
     * Пустой список или `null` удаляет все связанные GIGAUSAGE-записи.
     *
     * @param agent обновляемый AI-агент.
     * @param rawValue список JIRA-ключей или URL.
     */
    open fun update(
        agent: AIAgentEntity,
        rawValue: Any?,
    ) {
        // текущая реализация
    }

    /**
     * Преобразует необработанное значение запроса в список
     * валидных GIGAUSAGE-ключей или URL.
     *
     * @param rawValue значение поля `gigausage`.
     * @return нормализованный список строк.
     * @throws AiBadRequestException если формат значения некорректен.
     */
    private fun parseGigausageValues(
        rawValue: Any?,
    ): List<String> {
        // текущая реализация
    }

    /**
     * Проверяет, является ли значение допустимым JIRA-ключом
     * или URL проекта GIGAUSAGE.
     *
     * @param gigausage проверяемое значение.
     * @return `true`, если значение соответствует допустимому формату.
     */
    private fun isValidGigausage(
        gigausage: String,
    ): Boolean {
        // текущая реализация
    }

    /**
     * Извлекает JIRA-ключ GIGAUSAGE из ключа или полного URL.
     *
     * @param gigausage JIRA-ключ или URL.
     * @return извлечённый JIRA-ключ.
     */
    private fun extractJiraKey(
        gigausage: String,
    ): String {
        // текущая реализация
    }

    /**
     * Удаляет существующие JIRA-записи, ключи которых
     * отсутствуют в текущем запросе.
     *
     * @param existingIssues существующие GIGAUSAGE-записи агента.
     * @param requestedKeys ключи, переданные в запросе.
     */
    private fun deleteIssuesMissingInRequest(
        existingIssues: List<JiraIssueEntity>,
        requestedKeys: Set<String>,
    ) {
        // текущая реализация
    }

    /**
     * Создаёт новые или обновляет существующие GIGAUSAGE-записи.
     *
     * @param agent обновляемый AI-агент.
     * @param existingIssues существующие JIRA-записи агента.
     * @param requestedValuesByKey значения запроса, сгруппированные по JIRA-ключу.
     */
    private fun createOrUpdateIssues(
        agent: AIAgentEntity,
        existingIssues: List<JiraIssueEntity>,
        requestedValuesByKey: Map<String, String>,
    ) {
        // текущая реализация
    }

    /**
     * Заполняет JIRA-ключ и URL сущности в зависимости
     * от формата переданного значения.
     *
     * @param jiraIssue обновляемая сущность JIRA-задачи.
     * @param gigausage JIRA-ключ или полный URL.
     */
    private fun updateJiraKeyAndUrl(
        jiraIssue: JiraIssueEntity,
        gigausage: String,
    ) {
        // текущая реализация
    }

    /**
     * Завершает обработку ошибкой некорректного значения `gigausage`.
     *
     * @throws AiBadRequestException всегда.
     */
    private fun throwWrongGigausageValue(): Nothing {
        // текущая реализация
    }
}
```
