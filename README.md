```java
@Query(
    value = """
        select distinct issue.agent_id as "agentId", upper(issue.jira_key) as "jiraKey"
        from jira_issue issue
        join ai_agent agent on agent.id = issue.agent_id
        where issue.type = 'initiative'
          and lower(issue.project) = 'crossgoal'
          and upper(issue.jira_key) in (:jiraKeys)
          and agent.disabled is not true

        union

        select agent.id as "agentId",
               substring(upper(agent.agent_jira_url) from '(CROSSGOAL-[0-9]+)') as "jiraKey"
        from ai_agent agent
        where substring(upper(agent.agent_jira_url) from '(CROSSGOAL-[0-9]+)') in (:jiraKeys)
          and agent.disabled is not true
    """,
    nativeQuery = true
)
fun findInitiativeReferences(@Param("jiraKeys") jiraKeys: Collection<String>): List<JiraUpdatedInitiativeReference>
```
