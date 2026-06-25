```java
select initiative_agent_type_id,
       metric_directory_id,
       period_month,
       count(*)
from initiative_metric_value
group by initiative_agent_type_id,
         metric_directory_id,
         period_month
having count(*) > 1;

<changeSet id="replace-unique-initiative-metric-value-by-period" author="KoptenkoMV">

    <dropUniqueConstraint
        tableName="initiative_metric_value"
        constraintName="uk_initiative_metric_value_agent_type_metric"/>

    <addUniqueConstraint
        tableName="initiative_metric_value"
        columnNames="initiative_agent_type_id, metric_directory_id, period_month"
        constraintName="uq_initiative_metric_value_type_metric_period"/>

    <rollback>
        <dropUniqueConstraint
            tableName="initiative_metric_value"
            constraintName="uq_initiative_metric_value_type_metric_period"/>

        <addUniqueConstraint
            tableName="initiative_metric_value"
            columnNames="initiative_agent_type_id, metric_directory_id"
            constraintName="uk_initiative_metric_value_agent_type_metric"/>
    </rollback>

</changeSet>
```
