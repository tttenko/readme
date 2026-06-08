```java

@Component
class GigausageIssueUpdater(
    private val jiraIssueRepository: JiraIssueRepository,
    private val messageProvider: MessageProvider,
) {

    fun update(
        agent: AIAgentEntity,
        rawValue: Any?,
    ) {
        val gigausageValues = parseGigausageValues(rawValue)

        val existingIssues =
            jiraIssueRepository.findByAgentIdAndTypeAndProject(
                agentId = agent.id,
                type = ISSUE_TYPE,
                project = PROJECT,
            )

        if (gigausageValues.isEmpty()) {
            jiraIssueRepository.deleteAll(existingIssues)
            return
        }

        /*
         * Ключ — jiraKey.
         * Значение — исходная строка из запроса:
         * либо GIGAUSAGE-123,
         * либо URL.
         *
         * associateBy также убирает дубли по jiraKey.
         */
        val requestedValuesByKey = gigausageValues.associateBy { gigausage ->
            extractJiraKey(gigausage)
        }

        deleteIssuesMissingInRequest(
            existingIssues = existingIssues,
            requestedKeys = requestedValuesByKey.keys,
        )

        createOrUpdateIssues(
            agent = agent,
            existingIssues = existingIssues,
            requestedValuesByKey = requestedValuesByKey,
        )
    }

    private fun parseGigausageValues(
        rawValue: Any?,
    ): List<String> {
        if (rawValue == null) {
            return emptyList()
        }

        if (rawValue !is List<*>) {
            throwWrongGigausageValue()
        }

        return rawValue.map { value ->
            val gigausage = (value as? String)?.trim()

            if (gigausage.isNullOrEmpty() || !isValidGigausage(gigausage)) {
                throwWrongGigausageValue()
            }

            gigausage
        }
    }

    private fun isValidGigausage(
        gigausage: String,
    ): Boolean {
        val isJiraKey = gigausage.startsWith(GIGAUSAGE_PREFIX)

        val isJiraUrl =
            gigausage.startsWith(JIRA_URL_PREFIX) &&
                gigausage
                    .split("/")
                    .any { pathPart ->
                        pathPart.startsWith(GIGAUSAGE_PREFIX)
                    }

        return isJiraKey || isJiraUrl
    }

    private fun extractJiraKey(
        gigausage: String,
    ): String {
        if (gigausage.startsWith(GIGAUSAGE_PREFIX)) {
            return gigausage
        }

        return gigausage
            .split("/")
            .first { pathPart ->
                pathPart.startsWith(GIGAUSAGE_PREFIX)
            }
    }

    private fun deleteIssuesMissingInRequest(
        existingIssues: List<JiraIssueEntity>,
        requestedKeys: Set<String>,
    ) {
        existingIssues
            .filter { existingIssue ->
                existingIssue.jiraKey !in requestedKeys
            }
            .forEach { issueToDelete ->
                jiraIssueRepository.delete(entity = issueToDelete)
            }
    }

    private fun createOrUpdateIssues(
        agent: AIAgentEntity,
        existingIssues: List<JiraIssueEntity>,
        requestedValuesByKey: Map<String, String>,
    ) {
        val existingIssuesByKey = existingIssues.associateBy { issue ->
            issue.jiraKey
        }

        requestedValuesByKey.forEach { (jiraKey, gigausage) ->
            val jiraIssue =
                existingIssuesByKey[jiraKey]
                    ?: JiraIssueEntity(
                        project = PROJECT,
                        type = ISSUE_TYPE,
                        agent = agent,
                    )

            jiraIssue.project = PROJECT
            jiraIssue.type = ISSUE_TYPE

            updateJiraKeyAndUrl(
                jiraIssue = jiraIssue,
                gigausage = gigausage,
            )

            jiraIssueRepository.save(entity = jiraIssue)
        }
    }

    private fun updateJiraKeyAndUrl(
        jiraIssue: JiraIssueEntity,
        gigausage: String,
    ) {
        if (gigausage.startsWith(GIGAUSAGE_PREFIX)) {
            jiraIssue.jiraKey = gigausage
            jiraIssue.jiraUrl = "$JIRA_BROWSE_URL/$gigausage"
            return
        }

        jiraIssue.jiraKey = extractJiraKey(gigausage)
        jiraIssue.jiraUrl = gigausage
    }

    private fun throwWrongGigausageValue(): Nothing {
        throw AiBadRequestException(
            message = messageProvider[WRONG_WRONG_GIGAUSAGE],
            errorCode = WRONG_WRONG_GIGAUSAGE,
        )
    }

    private companion object {
        const val PROJECT = "gigausage"
        const val ISSUE_TYPE = "initiative"

        const val GIGAUSAGE_PREFIX = "GIGAUSAGE"

        const val JIRA_URL_PREFIX =
            "https://jira.sberbank.ru/"

        const val JIRA_BROWSE_URL =
            "https://jira.sberbank.ru/browse"
    }
}
```
