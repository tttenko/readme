```java
/**
 * Загружает email пользователей АУЕ, указанных в agent_contact.user_id.
 *
 * Все уникальные userId собираются заранее и запрашиваются одним bulk-вызовом,
 * чтобы не выполнять отдельный запрос в АУЕ для каждого контакта инициативы.
 */
private fun loadUserEmailsById(agents: List<AIAgentEntity>): Map<Long, String> {
    val userIds = agents.asSequence()
        .flatMap { it.agentContact.asSequence() }
        .mapNotNull { it.userId }
        .toSet()

    if (userIds.isEmpty()) return emptyMap()

    return authFeignClient.getUsers(ids = userIds, companyId = null)
        .body
        .orEmpty()
        .mapNotNull { user ->
            val email = user.email?.takeIf { it.isNotBlank() } ?: return@mapNotNull null
            user.id to email
        }
        .toMap()
}
```
