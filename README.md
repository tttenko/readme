```java
<changeSet id="Add unique constraint initiative_metric_value" author="KoptenkoMV">

    <addUniqueConstraint
        tableName="initiative_metric_value"
        columnNames="initiative_agent_type_id, metric_directory_id"
        constraintName="uk_initiative_metric_value_agent_type_metric"/>

    <rollback>
        <dropUniqueConstraint
            tableName="initiative_metric_value"
            constraintName="uk_initiative_metric_value_agent_type_metric"/>
    </rollback>

</changeSet>
```
