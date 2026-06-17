```java
<changeSet id="Create metrics_directory table" author="agenshirshov@sberbank.ru">

    <createTable tableName="metrics_directory"
                 remarks="Справочник метрик">

        <column name="id"
                type="UUID"
                remarks="ID метрики">
            <constraints primaryKey="true"
                         nullable="false"/>
        </column>

        <column name="name"
                type="VARCHAR(120)"
                remarks="Наименование метрики">
            <constraints nullable="false"/>
        </column>

        <column name="unit"
                type="VARCHAR(50)"
                remarks="Единица измерения">
            <constraints nullable="false"/>
        </column>

        <column name="direction"
                type="VARCHAR(50)"
                remarks="Направление оценки">
            <constraints nullable="false"/>
        </column>

        <column name="description"
                type="TEXT"
                remarks="Формула или описание метрики">
            <constraints nullable="false"/>
        </column>

        <column name="frequency"
                type="VARCHAR(50)"
                remarks="Периодичность сдачи">
            <constraints nullable="false"/>
        </column>

        <column name="copilot_applicability"
                type="BOOLEAN"
                remarks="Метрика доступна для использования в Copilot-агенте"/>

        <column name="autonomous_applicability"
                type="BOOLEAN"
                remarks="Метрика доступна для использования в автономном агенте"/>

        <column name="requires_appeals_work"
                type="BOOLEAN"
                remarks="Метрика доступна для использования в агенте по работе с обращениями"/>

        <column name="is_active"
                type="BOOLEAN"
                remarks="Активна или деактивирована метрика"/>

        <column name="updated_by"
                type="VARCHAR(255)"
                remarks="Идентификатор пользователя, внесшего изменения">
            <constraints nullable="false"/>
        </column>

        <column name="updated_at"
                type="TIMESTAMP"
                remarks="Дата и время изменения">
            <constraints nullable="false"/>
        </column>

    </createTable>

    <rollback>
        <dropTable tableName="metrics_directory"/>
    </rollback>

</changeSet>

<changeSet id="Create initiative_metric_type table" author="agenshirshov@sberbank.ru">

    <createTable tableName="initiative_metric_type"
                 remarks="Тип агента, прикрепленный к инициативе">

        <column name="id"
                type="BIGINT"
                autoIncrement="true"
                remarks="ID связи">
            <constraints primaryKey="true"
                         nullable="false"/>
        </column>

        <column name="ai_agent_id"
                type="BIGINT"
                remarks="Ссылка на инициативу">
            <constraints nullable="false"/>
        </column>

        <column name="agent_type"
                type="VARCHAR(50)"
                remarks="Тип агента">
            <constraints nullable="false"/>
        </column>

    </createTable>

    <addForeignKeyConstraint
            baseTableName="initiative_metric_type"
            baseColumnNames="ai_agent_id"
            constraintName="initiative_metric_type_AI_AGENT_ID_FK"
            referencedTableName="ai_agent"
            referencedColumnNames="id"/>

    <createIndex tableName="initiative_metric_type"
                 indexName="idx_initiative_metric_type_ai_agent_id">
        <column name="ai_agent_id"/>
    </createIndex>

    <rollback>
        <dropTable tableName="initiative_metric_type"/>
    </rollback>

</changeSet>

<changeSet id="Create initiative_metric_value table" author="agenshirshov@sberbank.ru">

    <createTable tableName="initiative_metric_value"
                 remarks="Значения по метрикам, которые были сданы для инициативы и типа агента">

        <column name="id"
                type="BIGINT"
                autoIncrement="true"
                remarks="ID записи">
            <constraints primaryKey="true"
                         nullable="false"/>
        </column>

        <column name="initiative_agent_type_id"
                type="BIGINT"
                remarks="Связка инициативы и типа агента">
            <constraints nullable="false"/>
        </column>

        <column name="metric_directory_id"
                type="UUID"
                remarks="Метрика">
            <constraints nullable="false"/>
        </column>

        <column name="period_month"
                type="DATE"
                remarks="Период сдачи метрики, первый день месяца">
            <constraints nullable="false"/>
        </column>

        <column name="metric_value"
                type="NUMERIC"
                remarks="Фактическое значение периода"/>

        <column name="target_value"
                type="NUMERIC"
                remarks="Плановое значение периода"/>

    </createTable>

    <addForeignKeyConstraint
            baseTableName="initiative_metric_value"
            baseColumnNames="initiative_agent_type_id"
            constraintName="initiative_metric_value_INITIATIVE_AGENT_TYPE_ID_FK"
            referencedTableName="initiative_metric_type"
            referencedColumnNames="id"/>

    <addForeignKeyConstraint
            baseTableName="initiative_metric_value"
            baseColumnNames="metric_directory_id"
            constraintName="initiative_metric_value_METRIC_DIRECTORY_ID_FK"
            referencedTableName="metrics_directory"
            referencedColumnNames="id"/>

    <createIndex tableName="initiative_metric_value"
                 indexName="idx_initiative_metric_value_initiative_agent_type_id">
        <column name="initiative_agent_type_id"/>
    </createIndex>

    <createIndex tableName="initiative_metric_value"
                 indexName="idx_initiative_metric_value_metric_directory_id">
        <column name="metric_directory_id"/>
    </createIndex>

    <rollback>
        <dropTable tableName="initiative_metric_value"/>
    </rollback>

</changeSet>


Entity: MetricsDirectoryEntity

@Entity
@Table(name = "metrics_directory")
open class MetricsDirectoryEntity(

    @Column(
        name = "name",
        length = 120,
        nullable = false,
    )
    var name: String? = null,

    @Column(
        name = "unit",
        length = 50,
        nullable = false,
    )
    var unit: String? = null,

    @Column(
        name = "direction",
        length = 50,
        nullable = false,
    )
    var direction: String? = null,

    @Column(
        name = "description",
        columnDefinition = "text",
        nullable = false,
    )
    var description: String? = null,

    @Column(
        name = "frequency",
        length = 50,
        nullable = false,
    )
    var frequency: String? = null,

    @Column(name = "copilot_applicability")
    var copilotApplicability: Boolean? = null,

    @Column(name = "autonomous_applicability")
    var autonomousApplicability: Boolean? = null,

    @Column(name = "requires_appeals_work")
    var requiresAppealsWork: Boolean? = null,

    @Column(name = "is_active")
    var active: Boolean? = null,

    @Column(
        name = "updated_by",
        nullable = false,
    )
    var updatedBy: String? = null,

    @Column(
        name = "updated_at",
        nullable = false,
    )
    var updatedAt: LocalDateTime = LocalDateTime.now(),

) : BasicUUIDEntity()

Entity: InitiativeMetricTypeEntity
@Entity
@Table(name = "initiative_metric_type")
open class InitiativeMetricTypeEntity(

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(
        name = "ai_agent_id",
        referencedColumnName = "id",
        nullable = false,
    )
    var aiAgent: AIAgentEntity? = null,

    @Column(
        name = "agent_type",
        length = 50,
        nullable = false,
    )
    var agentType: String? = null,

    @OneToMany(
        targetEntity = InitiativeMetricValueEntity::class,
        mappedBy = "initiativeMetricType",
        cascade = [CascadeType.ALL],
        orphanRemoval = true,
    )
    var metricValues: MutableList<InitiativeMetricValueEntity> = mutableListOf(),

) : BasicLongEntity()
Entity: InitiativeMetricValueEntity
@Entity
@Table(name = "initiative_metric_value")
open class InitiativeMetricValueEntity(

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(
        name = "initiative_agent_type_id",
        referencedColumnName = "id",
        nullable = false,
    )
    var initiativeMetricType: InitiativeMetricTypeEntity? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(
        name = "metric_directory_id",
        referencedColumnName = "id",
        nullable = false,
    )
    var metricDirectory: MetricsDirectoryEntity? = null,

    @Column(
        name = "period_month",
        nullable = false,
    )
    var periodMonth: LocalDate? = null,

    @Column(name = "metric_value")
    var metricValue: BigDecimal? = null,

    @Column(name = "target_value")
    var targetValue: BigDecimal? = null,

) : BasicLongEntity()

```
