```java

@Test
fun `should keep created initiative and monitoring epic when monitoring task search fails`() {
    jiraStub.failMonitoringTaskSearch()

    jiraNewInitiativeImportService.importNewInitiatives()

    val agent = agentRepository.findFirstByAgentId(NEW_INITIATIVE_KEY)

    assertThat(agent)
        .withFailMessage("New Jira initiative must remain persisted after monitoring Task search error")
        .isNotNull

    requireNotNull(agent)

    assertThat(agent.agentStatus?.code).isEqualTo(ANALYSIS_STATUS_CODE)
    assertThat(agent.jiraFromStatus).isEqualTo(JIRA_FROM_STATUS_ERROR)
    assertThat(agent.jiraUpdated).isNotNull

    val initiativeIssues = jiraIssueRepository.findByAgentIdAndTypeAndProject(
        agent.id,
        JiraIssueType.initiative.name,
        CROSSGOAL_PROJECT,
    )

    assertThat(initiativeIssues)
        .anySatisfy { issue ->
            assertThat(issue.jiraKey).isEqualTo(NEW_INITIATIVE_KEY)
        }

    val monitoringEpicIssues = jiraIssueRepository.findByAgentIdAndTypeAndProject(
        agent.id,
        JiraIssueType.epic.name,
        CROSSGOAL_PROJECT,
    )

    assertThat(monitoringEpicIssues).hasSize(1)
    assertThat(monitoringEpicIssues.single().jiraKey).isEqualTo(MONITORING_EPIC_KEY)

    val monitoringTaskIssues = jiraIssueRepository.findAllByAgentIdAndType(
        agent.id,
        JiraIssueType.task.name,
    )

    assertThat(monitoringTaskIssues).isEmpty()

    val statusSlas = agentStatusSlaRepository.findAllByAiAgentId(agent.id)

    assertThat(statusSlas).isEmpty()

    val qualityGates = jdbcTemplate.queryForMap(
        """
        select quality_gate_code, state
        from agent_quality_gate
        where ai_agent_id = ?
        """.trimIndent(),
        agent.id,
    )

    assertThat(qualityGates).isNotEmpty
    assertThat(qualityGates.values).allMatch { state -> state == "unchecked" }

    assertThat(jiraStub.monitoringTaskSearchRequestCount()).isGreaterThanOrEqualTo(1)
}
```
