```java
/**
 * Определяет CROSSGOAL-ключ родительской инициативы.
 *
 * Основной источник — сохранённая связь jira_issue типа initiative.
 * Это позволяет обрабатывать Task инициатив, созданных из Пульта,
 * даже если agentJiraUrl и agentId не содержат CROSSGOAL-ключ.
 *
 * При отсутствии ключа в сохранённых связях использует поля агента.
 * Несколько разных ключей считаются неоднозначной связью:
 * такая инициатива пропускается через существующую обработку UNAVAILABLE.
 */
private fun resolveInitiativeJiraKey(agent: AIAgentEntity): String? {
    val savedInitiativeKeys = jiraIssueRepository.findByAgentIdAndTypeAndProject(
        agentId = agent.id,
        type = JiraIssueType.initiative.name,
        project = "crossgoal"
    ).mapNotNull { initiativeIssue ->
        jiraIssueKeyExtractor.extractCrossgoalKey(initiativeIssue.jiraKey)
            ?: jiraIssueKeyExtractor.extractCrossgoalKey(initiativeIssue.jiraUrl)
    }.toSet()

    if (savedInitiativeKeys.size > 1) {
        log.warn(
            "Cannot determine unique CROSSGOAL initiative key: agentId={}, jiraKeys={}",
            agent.id, savedInitiativeKeys
        )
        return null
    }

    val savedInitiativeKey = savedInitiativeKeys.singleOrNull()
    if (savedInitiativeKey != null) {
        log.debug(
            "Resolved CROSSGOAL initiative key from jira_issue: agentId={}, jiraKey={}",
            agent.id, savedInitiativeKey
        )
        return savedInitiativeKey
    }

    return jiraIssueKeyExtractor.extractCrossgoalKey(agent.agentJiraUrl)
        ?: jiraIssueKeyExtractor.extractCrossgoalKey(agent.agentId)
}
```
