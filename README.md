```java

<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd">

    <changeSet id="Add cascade delete for initiative metric relations" author="KoptenkoMV">

        <!-- initiative_metric_type -> initiative_metric_value -->

        <dropForeignKeyConstraint
                baseTableName="initiative_metric_value"
                constraintName="initiative_metric_value_INITIATIVE_AGENT_TYPE_ID_FK"/>

        <addForeignKeyConstraint
                baseTableName="initiative_metric_value"
                baseColumnNames="initiative_agent_type_id"
                constraintName="initiative_metric_value_INITIATIVE_AGENT_TYPE_ID_FK"
                referencedTableName="initiative_metric_type"
                referencedColumnNames="id"
                onDelete="CASCADE"/>


        <!-- initiative_metric_type -> initiative_metric_assignment -->

        <dropForeignKeyConstraint
                baseTableName="initiative_metric_assignment"
                constraintName="initiative_metric_assignment_INITIATIVE_AGENT_TYPE_ID_FK"/>

        <addForeignKeyConstraint
                baseTableName="initiative_metric_assignment"
                baseColumnNames="initiative_agent_type_id"
                constraintName="initiative_metric_assignment_INITIATIVE_AGENT_TYPE_ID_FK"
                referencedTableName="initiative_metric_type"
                referencedColumnNames="id"
                onDelete="CASCADE"/>


        <!-- initiative_metric_assignment -> metric_applicability_request -->

        <dropForeignKeyConstraint
                baseTableName="metric_applicability_request"
                constraintName="metric_applicability_request_ASSIGNMENT_ID_FK"/>

        <addForeignKeyConstraint
                baseTableName="metric_applicability_request"
                baseColumnNames="initiative_metric_assignment_id"
                constraintName="metric_applicability_request_ASSIGNMENT_ID_FK"
                referencedTableName="initiative_metric_assignment"
                referencedColumnNames="id"
                onDelete="CASCADE"/>


        <!-- metric_applicability_request -> metric_applicability_history -->

        <dropForeignKeyConstraint
                baseTableName="metric_applicability_history"
                constraintName="metric_applicability_history_REQUEST_ID_FK"/>

        <addForeignKeyConstraint
                baseTableName="metric_applicability_history"
                baseColumnNames="metric_applicability_request_id"
                constraintName="metric_applicability_history_REQUEST_ID_FK"
                referencedTableName="metric_applicability_request"
                referencedColumnNames="id"
                onDelete="CASCADE"/>


        <rollback>

            <dropForeignKeyConstraint
                    baseTableName="metric_applicability_history"
                    constraintName="metric_applicability_history_REQUEST_ID_FK"/>

            <addForeignKeyConstraint
                    baseTableName="metric_applicability_history"
                    baseColumnNames="metric_applicability_request_id"
                    constraintName="metric_applicability_history_REQUEST_ID_FK"
                    referencedTableName="metric_applicability_request"
                    referencedColumnNames="id"/>


            <dropForeignKeyConstraint
                    baseTableName="metric_applicability_request"
                    constraintName="metric_applicability_request_ASSIGNMENT_ID_FK"/>

            <addForeignKeyConstraint
                    baseTableName="metric_applicability_request"
                    baseColumnNames="initiative_metric_assignment_id"
                    constraintName="metric_applicability_request_ASSIGNMENT_ID_FK"
                    referencedTableName="initiative_metric_assignment"
                    referencedColumnNames="id"/>


            <dropForeignKeyConstraint
                    baseTableName="initiative_metric_assignment"
                    constraintName="initiative_metric_assignment_INITIATIVE_AGENT_TYPE_ID_FK"/>

            <addForeignKeyConstraint
                    baseTableName="initiative_metric_assignment"
                    baseColumnNames="initiative_agent_type_id"
                    constraintName="initiative_metric_assignment_INITIATIVE_AGENT_TYPE_ID_FK"
                    referencedTableName="initiative_metric_type"
                    referencedColumnNames="id"/>


            <dropForeignKeyConstraint
                    baseTableName="initiative_metric_value"
                    constraintName="initiative_metric_value_INITIATIVE_AGENT_TYPE_ID_FK"/>

            <addForeignKeyConstraint
                    baseTableName="initiative_metric_value"
                    baseColumnNames="initiative_agent_type_id"
                    constraintName="initiative_metric_value_INITIATIVE_AGENT_TYPE_ID_FK"
                    referencedTableName="initiative_metric_type"
                    referencedColumnNames="id"/>

        </rollback>

    </changeSet>

</databaseChangeLog>
```
