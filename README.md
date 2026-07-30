```java
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://www.liquibase.org/xml/ns/dbchangelog
            http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd">

    <changeSet
            id="add-index-agent-contact-user-id-agent-id"
            author="KoptenkoMV">

        <createIndex
                tableName="agent_contact"
                indexName="idx_agent_contact_user_id_agent_id">

            <column name="user_id"/>
            <column name="agent_id"/>
        </createIndex>

        <rollback>
            <dropIndex
                    tableName="agent_contact"
                    indexName="idx_agent_contact_user_id_agent_id"/>
        </rollback>
    </changeSet>

    <changeSet
            id="add-index-agent-status-sla-deviations"
            author="KoptenkoMV">

        <createIndex
                tableName="agent_status_sla"
                indexName="idx_agent_status_sla_agent_completed_planned">

            <column name="ai_agent_id"/>
            <column name="completed_date"/>
            <column name="planned_date"/>
        </createIndex>

        <rollback>
            <dropIndex
                    tableName="agent_status_sla"
                    indexName="idx_agent_status_sla_agent_completed_planned"/>
        </rollback>
    </changeSet>

    <changeSet
            id="add-index-jira-issue-agent-project-key"
            author="KoptenkoMV">

        <createIndex
                tableName="jira_issue"
                indexName="idx_jira_issue_agent_project_key">

            <column name="agent_id"/>
            <column name="project"/>
            <column name="jira_key"/>
        </createIndex>

        <rollback>
            <dropIndex
                    tableName="jira_issue"
                    indexName="idx_jira_issue_agent_project_key"/>
        </rollback>
    </changeSet>

    <changeSet
            id="add-index-initiative-metric-type-agent-type"
            author="KoptenkoMV">

        <createIndex
                tableName="initiative_metric_type"
                indexName="idx_initiative_metric_type_agent_id_type">

            <column name="ai_agent_id"/>
            <column name="agent_type"/>
        </createIndex>

        <rollback>
            <dropIndex
                    tableName="initiative_metric_type"
                    indexName="idx_initiative_metric_type_agent_id_type"/>
        </rollback>
    </changeSet>

</databaseChangeLog>
  ```
