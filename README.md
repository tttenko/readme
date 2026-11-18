```java


private fun getCrossgoal(agent: AIAgentEntity): String {
    return agent.jiraIssues
        .firstOrNull { it.project.equals("crossgoal", ignoreCase = true) && it.type.equals("initiative", ignoreCase = true) }
        ?.jiraKey
        ?.takeIf { it.isNotBlank() }
        ?: throw IllegalStateException("Не найден CROSSGOAL для инициативы ${agent.id}")
}

```
