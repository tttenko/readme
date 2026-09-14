```java
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd">

    <!-- ========================================================= -->
    <!-- initiative_metric_assignment                              -->
    <!-- ========================================================= -->

    <changeSet id="Create initiative_metric_assignment table" author="KoptenkoMV">

        <createTable
                tableName="initiative_metric_assignment"
                remarks="Привязка метрики к типу агента инициативы">

            <column
                    name="id"
                    type="BIGINT"
                    autoIncrement="true"
                    remarks="ID постоянной физической связи">
                <constraints
                        primaryKey="true"
                        nullable="false"/>
            </column>

            <column
                    name="initiative_agent_type_id"
                    type="BIGINT"
                    remarks="Ссылка на тип агента инициативы">
                <constraints nullable="false"/>
            </column>

            <column
                    name="metric_id"
                    type="UUID"
                    remarks="Ссылка на метрику">
                <constraints nullable="false"/>
            </column>

            <column
                    name="applicability_status"
                    type="VARCHAR(50)"
                    remarks="Текущий статус применимости метрики">
                <constraints nullable="false"/>
            </column>

        </createTable>

        <addForeignKeyConstraint
                baseTableName="initiative_metric_assignment"
                baseColumnNames="initiative_agent_type_id"
                constraintName="initiative_metric_assignment_INITIATIVE_AGENT_TYPE_ID_FK"
                referencedTableName="initiative_metric_type"
                referencedColumnNames="id"/>

        <addForeignKeyConstraint
                baseTableName="initiative_metric_assignment"
                baseColumnNames="metric_id"
                constraintName="initiative_metric_assignment_METRIC_ID_FK"
                referencedTableName="metrics_directory"
                referencedColumnNames="id"/>

        <addUniqueConstraint
                tableName="initiative_metric_assignment"
                columnNames="initiative_agent_type_id, metric_id"
                constraintName="initiative_metric_assignment_agent_type_metric_UK"/>

        <createIndex
                tableName="initiative_metric_assignment"
                indexName="idx_initiative_metric_assignment_agent_type_id">
            <column name="initiative_agent_type_id"/>
        </createIndex>

        <createIndex
                tableName="initiative_metric_assignment"
                indexName="idx_initiative_metric_assignment_metric_id">
            <column name="metric_id"/>
        </createIndex>

        <rollback>
            <dropTable tableName="initiative_metric_assignment"/>
        </rollback>

    </changeSet>


    <!-- ========================================================= -->
    <!-- metric_applicability_request                              -->
    <!-- ========================================================= -->

    <changeSet id="Create metric_applicability_request table" author="KoptenkoMV">

        <createTable
                tableName="metric_applicability_request"
                remarks="Заявка на изменение применимости метрики">

            <column
                    name="id"
                    type="BIGINT"
                    autoIncrement="true"
                    remarks="ID заявки">
                <constraints
                        primaryKey="true"
                        nullable="false"/>
            </column>

            <column
                    name="initiative_metric_assignment_id"
                    type="BIGINT"
                    remarks="Ссылка на привязку метрики к инициативе">
                <constraints nullable="false"/>
            </column>

            <column
                    name="status"
                    type="VARCHAR(50)"
                    remarks="Статус заявки">
                <constraints nullable="false"/>
            </column>

            <column
                    name="comment"
                    type="VARCHAR(1000)"
                    remarks="Обоснование создания заявки">
                <constraints nullable="false"/>
            </column>

            <column
                    name="resume_period"
                    type="DATE"
                    remarks="Период автоматического возврата применимости метрики"/>

            <column
                    name="is_visible_in_office"
                    type="BOOLEAN"
                    remarks="Признак отображения заявки в очереди Офиса">
                <constraints nullable="false"/>
            </column>

            <column
                    name="effective_from_period"
                    type="DATE"
                    remarks="Период начала неприменимости метрики">
                <constraints nullable="false"/>
            </column>

            <column
                    name="effective_to_period"
                    type="DATE"
                    remarks="Период окончания неприменимости метрики"/>

            <column
                    name="created_by"
                    type="BIGINT"
                    remarks="ID пользователя, создавшего заявку">
                <constraints nullable="false"/>
            </column>

            <column
                    name="decision_by"
                    type="BIGINT"
                    remarks="ID пользователя, принявшего решение по заявке"/>

            <column
                    name="created_at"
                    type="DATE"
                    remarks="Дата создания заявки">
                <constraints nullable="false"/>
            </column>

            <column
                    name="updated_at"
                    type="TIMESTAMP"
                    remarks="Дата и время последнего изменения заявки">
                <constraints nullable="false"/>
            </column>

        </createTable>

        <addForeignKeyConstraint
                baseTableName="metric_applicability_request"
                baseColumnNames="initiative_metric_assignment_id"
                constraintName="metric_applicability_request_ASSIGNMENT_ID_FK"
                referencedTableName="initiative_metric_assignment"
                referencedColumnNames="id"/>

        <createIndex
                tableName="metric_applicability_request"
                indexName="idx_metric_applicability_request_assignment_id">
            <column name="initiative_metric_assignment_id"/>
        </createIndex>

        <createIndex
                tableName="metric_applicability_request"
                indexName="idx_metric_applicability_request_office_status">
            <column name="is_visible_in_office"/>
            <column name="status"/>
            <column name="created_at"/>
            <column name="id"/>
        </createIndex>

        <!--
            Для одного assignment одновременно может существовать
            только одна заявка со статусом PENDING.
            Обычный addUniqueConstraint здесь не подходит,
            поэтому используем partial unique index PostgreSQL.
        -->
        <sql>
            CREATE UNIQUE INDEX uq_metric_applicability_request_pending_assignment
            ON metric_applicability_request (initiative_metric_assignment_id)
            WHERE status = 'PENDING';
        </sql>

        <rollback>
            <dropTable tableName="metric_applicability_request"/>
        </rollback>

    </changeSet>


    <!-- ========================================================= -->
    <!-- metric_applicability_history                              -->
    <!-- ========================================================= -->

    <changeSet id="Create metric_applicability_history table" author="KoptenkoMV">

        <createTable
                tableName="metric_applicability_history"
                remarks="История действий по заявке на изменение применимости метрики">

            <column
                    name="id"
                    type="BIGINT"
                    autoIncrement="true"
                    remarks="ID записи истории">
                <constraints
                        primaryKey="true"
                        nullable="false"/>
            </column>

            <column
                    name="metric_applicability_request_id"
                    type="BIGINT"
                    remarks="Ссылка на заявку">
                <constraints nullable="false"/>
            </column>

            <column
                    name="action"
                    type="VARCHAR(50)"
                    remarks="Выполненное действие">
                <constraints nullable="false"/>
            </column>

            <column
                    name="created_by"
                    type="VARCHAR(255)"
                    remarks="Пользователь или SYSTEM, выполнивший действие">
                <constraints nullable="false"/>
            </column>

            <column
                    name="comment"
                    type="VARCHAR(1000)"
                    remarks="Комментарий действия"/>

            <column
                    name="created_at"
                    type="DATE"
                    remarks="Дата выполнения действия">
                <constraints nullable="false"/>
            </column>

        </createTable>

        <addForeignKeyConstraint
                baseTableName="metric_applicability_history"
                baseColumnNames="metric_applicability_request_id"
                constraintName="metric_applicability_history_REQUEST_ID_FK"
                referencedTableName="metric_applicability_request"
                referencedColumnNames="id"/>

        <createIndex
                tableName="metric_applicability_history"
                indexName="idx_metric_applicability_history_request_id">
            <column name="metric_applicability_request_id"/>
            <column name="created_at"/>
            <column name="id"/>
        </createIndex>

        <rollback>
            <dropTable tableName="metric_applicability_history"/>
        </rollback>

    </changeSet>

</databaseChangeLog>


enum class MetricApplicabilityStatus {
    ACTIVE,
    PENDING,
    NOT_APPLICABLE,
}
enum class MetricApplicabilityRequestStatus {
    PENDING,
    APPROVED,
    REJECTED,
}
enum class MetricApplicabilityAction {
    REQUEST_CREATED,
    APPROVED,
    REJECTED,
    CANCEL_DECISION,
    RESTORED,
    RESUME_PERIOD,
}


@Entity
@Table(
    name = "initiative_metric_assignment",
    uniqueConstraints = [
        UniqueConstraint(
            name = "initiative_metric_assignment_agent_type_metric_UK",
            columnNames = [
                "initiative_agent_type_id",
                "metric_id",
            ],
        ),
    ],
)
class InitiativeMetricAssignmentEntity(

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(
        name = "initiative_agent_type_id",
        referencedColumnName = "id",
        nullable = false,
    )
    var initiativeMetricType: InitiativeMetricTypeEntity? = null,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(
        name = "metric_id",
        referencedColumnName = "id",
        nullable = false,
    )
    var metric: MetricsDirectoryEntity? = null,

    @Enumerated(EnumType.STRING)
    @Column(
        name = "applicability_status",
        length = 50,
        nullable = false,
    )
    var applicabilityStatus: MetricApplicabilityStatus = MetricApplicabilityStatus.ACTIVE,

    @OneToMany(
        mappedBy = "initiativeMetricAssignment",
        fetch = FetchType.LAZY,
    )
    var requests: MutableList<MetricApplicabilityRequestEntity> = mutableListOf(),

    ) : BasicLongEntity()


@Entity
@Table(name = "metric_applicability_request")
class MetricApplicabilityRequestEntity(

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(
        name = "initiative_metric_assignment_id",
        referencedColumnName = "id",
        nullable = false,
    )
    var initiativeMetricAssignment: InitiativeMetricAssignmentEntity? = null,

    @Enumerated(EnumType.STRING)
    @Column(
        name = "status",
        length = 50,
        nullable = false,
    )
    var status: MetricApplicabilityRequestStatus = MetricApplicabilityRequestStatus.PENDING,

    @Column(
        name = "comment",
        length = 1000,
        nullable = false,
    )
    var comment: String = "",

    @Column(name = "resume_period")
    var resumePeriod: LocalDate? = null,

    @Column(
        name = "is_visible_in_office",
        nullable = false,
    )
    var isVisibleInOffice: Boolean = true,

    @Column(
        name = "effective_from_period",
        nullable = false,
    )
    var effectiveFromPeriod: LocalDate? = null,

    @Column(name = "effective_to_period")
    var effectiveToPeriod: LocalDate? = null,

    @Column(
        name = "created_by",
        nullable = false,
    )
    var createdBy: Long = 0L,

    @Column(name = "decision_by")
    var decisionBy: Long? = null,

    @Column(
        name = "created_at",
        nullable = false,
    )
    var createdAt: LocalDate? = null,

    @Column(
        name = "updated_at",
        nullable = false,
    )
    var updatedAt: LocalDateTime? = null,

    @OneToMany(
        mappedBy = "metricApplicabilityRequest",
        fetch = FetchType.LAZY,
    )
    var history: MutableList<MetricApplicabilityHistoryEntity> = mutableListOf(),

    ) : BasicLongEntity() {

    @PrePersist
    protected fun onCreate() {
        createdAt = LocalDate.now()
        updatedAt = LocalDateTime.now()
    }

    @PreUpdate
    protected fun onUpdate() {
        updatedAt = LocalDateTime.now()
    }
}

@Entity
@Table(name = "metric_applicability_history")
class MetricApplicabilityHistoryEntity(

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(
        name = "metric_applicability_request_id",
        referencedColumnName = "id",
        nullable = false,
    )
    var metricApplicabilityRequest: MetricApplicabilityRequestEntity? = null,

    @Enumerated(EnumType.STRING)
    @Column(
        name = "action",
        length = 50,
        nullable = false,
    )
    var action: MetricApplicabilityAction? = null,

    @Column(
        name = "created_by",
        length = 255,
        nullable = false,
    )
    var createdBy: String = "",

    @Column(
        name = "comment",
        length = 1000,
    )
    var comment: String? = null,

    @Column(
        name = "created_at",
        nullable = false,
    )
    var createdAt: LocalDate? = null,

    ) : BasicLongEntity() {

    @PrePersist
    protected fun onCreate() {
        createdAt = LocalDate.now()
    }
}


@Repository
interface InitiativeMetricAssignmentRepository :
    JpaRepository<InitiativeMetricAssignmentEntity, Long>
@Repository
interface MetricApplicabilityRequestRepository :
    JpaRepository<MetricApplicabilityRequestEntity, Long>
@Repository
interface MetricApplicabilityHistoryRepository :
    JpaRepository<MetricApplicabilityHistoryEntity, Long>
```
