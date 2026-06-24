```java
@Modifying
    @Query(
        value = """
            delete from InitiativeMetricValueEntity metricValue
            where metricValue.initiativeMetricType.id in :initiativeMetricTypeIds
        """
    )
    fun deleteAllByInitiativeMetricTypeIds(
        @Param("initiativeMetricTypeIds")
        initiativeMetricTypeIds: Set<Long>,
    )

<changeSet id="add-unique-initiative-metric-value-by-period" author="your_name">
    <addUniqueConstraint
        tableName="initiative_metric_value"
        columnNames="initiative_agent_type_id, metric_directory_id, period_month"
        constraintName="uq_initiative_metric_value_type_metric_period"/>
</changeSet>
```
