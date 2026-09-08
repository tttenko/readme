```java
@Test
fun `should use contact email when agent contact has no user`() {
    val contact = ContactEntity().also {
        it.email = "contact@sberbank.ru"
    }

    val entity = AIAgentEntity().also {
        it.id = 1L
        it.agentContact = mutableListOf(
            AgentContactEntity(
                agent = it,
                type = "customer",
                contact = contact,
                userId = null,
            )
        )
    }

    val result = mapper.toInitiativeRegistryExcelExportModel(entity = entity)

    assertEquals("contact@sberbank.ru", result.contactEmails)
}

@Test
fun `should use AUE email when agent contact has user`() {
    val entity = AIAgentEntity().also {
        it.id = 1L
        it.agentContact = mutableListOf(
            AgentContactEntity(
                agent = it,
                type = "customer",
                contact = null,
                userId = 515L,
            )
        )
    }

    val result = mapper.toInitiativeRegistryExcelExportModel(
        entity = entity,
        userEmailsById = mapOf(515L to "user@sberbank.ru"),
    )

    assertEquals("user@sberbank.ru", result.contactEmails)
}

И я бы обязательно добавил третий, потому что он проверяет именно приоритет из нового требования:

@Test
fun `should prefer AUE email when agent contact has both user and contact`() {
    val contact = ContactEntity().also {
        it.email = "contact@sberbank.ru"
    }

    val entity = AIAgentEntity().also {
        it.id = 1L
        it.agentContact = mutableListOf(
            AgentContactEntity(
                agent = it,
                type = "customer",
                contact = contact,
                userId = 515L,
            )
        )
    }

    val result = mapper.toInitiativeRegistryExcelExportModel(
        entity = entity,
        userEmailsById = mapOf(515L to "user@sberbank.ru"),
    )

    assertEquals("user@sberbank.ru", result.contactEmails)
}
```
