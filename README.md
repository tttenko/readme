```java
/**
 * Создает запись в jira_change для последующей асинхронной синхронизации
 * изменения состояния вехи агента с Jira.
 *
 * Метод не выполняет синхронный вызов Jira. Он только формирует payload
 * с кодом вехи, новым статусом и ключом связанной Jira-задачи,
 * после чего сохраняет запись для обработки шедулером.
 */

 @Test
    fun `createQualityGateChange should save quality gate jira change with payload`() {
        // Given
        val aiAgent =
            AIAgentEntity().also {
                it.id = 1L
            }

        val linkedQualityGateJiraIssue =
            JiraIssueEntity().also {
                it.jiraKey = "TEST-1"
            }

        val savedJiraChangeSlot =
            slot<JiraChangeEntity>()

        every {
            jiraChangeRepository.save(
                capture(savedJiraChangeSlot),
            )
        } answers {
            firstArg()
        }

        // When
        jiraChangeCreator.createQualityGateChange(
            aiAgent = aiAgent,
            qualityGateCode = "GATE1",
            qualityGateState = QualityGateState.checked,
            linkedQualityGateJiraIssue = linkedQualityGateJiraIssue,
        )

        // Then
        verify(exactly = 1) {
            jiraChangeRepository.save(
                any<JiraChangeEntity>(),
            )
        }

        val savedJiraChange =
            savedJiraChangeSlot.captured

        assertEquals(
            aiAgent,
            savedJiraChange.agent,
        )

        assertEquals(
            "quality_gate",
            savedJiraChange.changeType,
        )

        assertNotNull(
            savedJiraChange.created,
        )

        assertEquals(
            "GATE1",
            savedJiraChange.payload?.get("qualityGateCode")?.asText(),
        )

        assertEquals(
            QualityGateState.checked.name,
            savedJiraChange.payload?.get("status")?.asText(),
        )

        assertEquals(
            "TEST-1",
            savedJiraChange.payload?.get("jiraKey")?.asText(),
        )
    }
```
