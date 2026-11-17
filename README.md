```java

import jakarta.persistence.*
import java.time.LocalDateTime

/**
 * Связанный AI-объект инициативы.
 * Хранит КЭ и дополнительные сведения об агенте или LLM-решении.
 */
@Entity
@Table(name = "initiative_meta_ai_object")
open class InitiativeMetaAiObjectEntity(

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "initiative_id", referencedColumnName = "id", nullable = false)
    var initiative: AIAgentEntity? = null,

    @Column(name = "name", length = 255)
    var name: String? = null,

    @Column(name = "is_agent")
    var isAgent: Boolean? = null,

    @Column(name = "gigausage", length = 255)
    var gigausage: String? = null,

    @Column(name = "meta_link", length = 255)
    var metaLink: String? = null,

    @Column(name = "ci_solution", columnDefinition = "VARCHAR")
    var ciSolution: String? = null,

    @Enumerated(EnumType.STRING)
    @Column(name = "source", nullable = false, length = 32)
    var source: InitiativeMetaAiObjectSource = InitiativeMetaAiObjectSource.MANUAL,

    @Column(name = "is_active", nullable = false)
    var isActive: Boolean = true,

    @Column(name = "created_by", columnDefinition = "VARCHAR")
    var createdBy: String? = null,

    @Column(name = "created_at", nullable = false, updatable = false)
    var createdAt: LocalDateTime? = null,

    @Column(name = "updated_by", columnDefinition = "VARCHAR")
    var updatedBy: String? = null,

    @Column(name = "updated_at", nullable = false)
    var updatedAt: LocalDateTime? = null

) : BasicLongEntity() {

    @PrePersist
    fun onCreate() {
        val now = LocalDateTime.now()
        if (createdAt == null) createdAt = now
        if (updatedAt == null) updatedAt = now
    }

    @PreUpdate
    fun onUpdate() {
        updatedAt = LocalDateTime.now()
    }
}


/**
 * Источник добавления связанного AI-объекта.
 */
enum class InitiativeMetaAiObjectSource {
    META,
    MANUAL
}


import jakarta.persistence.*
import java.time.LocalDateTime

/**
 * История фактических изменений AI-объектов инициативы.
 * Одно событие формируется на инициативу в рамках файловой загрузки.
 */
@Entity
@Table(name = "initiative_ai_object_history")
open class InitiativeAiObjectHistoryEntity(

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "ai_agent_id", referencedColumnName = "id", nullable = false)
    var initiative: AIAgentEntity? = null,

    @Column(name = "file_upload_id")
    var fileUploadId: Long? = null,

    @Column(name = "changed_objects_count", nullable = false)
    var changedObjectsCount: Int = 0,

    @Enumerated(EnumType.STRING)
    @Column(name = "event_type", nullable = false, length = 32)
    var eventType: InitiativeAiObjectEventType = InitiativeAiObjectEventType.FILE_IMPORT,

    @Column(name = "created_by", columnDefinition = "VARCHAR")
    var createdBy: String? = null,

    @Column(name = "created_by_name", length = 500)
    var createdByName: String? = null,

    @Column(name = "created_at", nullable = false, updatable = false)
    var createdAt: LocalDateTime? = null

) : BasicLongEntity() {

    @PrePersist
    fun onCreate() {
        if (createdAt == null) createdAt = LocalDateTime.now()
    }
}


/**
 * Тип события изменения AI-объектов.
 */
enum class InitiativeAiObjectEventType {
    FILE_IMPORT
}


<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd">

    <changeSet id="create-initiative-meta-ai-object" author="PolitovAY">

        <createTable tableName="initiative_meta_ai_object"
                     remarks="Связанные AI-объекты инициатив">

            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true"
                             primaryKeyName="pk_initiative_meta_ai_object"
                             nullable="false"/>
            </column>

            <column name="initiative_id" type="BIGINT"
                    remarks="FK на ai_agent.id">
                <constraints nullable="false"/>
            </column>

            <column name="name" type="VARCHAR(255)"
                    remarks="Наименование"/>

            <column name="is_agent" type="BOOLEAN"
                    remarks="Агент или LLM-решение"/>

            <column name="gigausage" type="VARCHAR(255)"
                    remarks="GigaUsage ID"/>

            <column name="meta_link" type="VARCHAR(255)"
                    remarks="Ссылка на Мета"/>

            <column name="ci_solution" type="VARCHAR"
                    remarks="КЭ решения"/>

            <column name="source" type="VARCHAR(32)"
                    remarks="Источник добавления META или MANUAL">
                <constraints nullable="false"/>
            </column>

            <column name="is_active" type="BOOLEAN"
                    defaultValueBoolean="true"
                    remarks="Признак активности связи">
                <constraints nullable="false"/>
            </column>

            <column name="created_by" type="VARCHAR"
                    remarks="ID автора"/>

            <column name="created_at" type="TIMESTAMP"
                    defaultValueComputed="CURRENT_TIMESTAMP"
                    remarks="Дата создания">
                <constraints nullable="false"/>
            </column>

            <column name="updated_by" type="VARCHAR"
                    remarks="ID автора последнего изменения"/>

            <column name="updated_at" type="TIMESTAMP"
                    defaultValueComputed="CURRENT_TIMESTAMP"
                    remarks="Дата изменения">
                <constraints nullable="false"/>
            </column>

        </createTable>

        <addForeignKeyConstraint
                baseTableName="initiative_meta_ai_object"
                baseColumnNames="initiative_id"
                constraintName="fk_initiative_meta_ai_object_initiative"
                referencedTableName="ai_agent"
                referencedColumnNames="id"/>

        <createIndex
                tableName="initiative_meta_ai_object"
                indexName="idx_initiative_meta_ai_object_initiative_ci">
            <column name="initiative_id"/>
            <column name="ci_solution"/>
        </createIndex>

        <createIndex
                tableName="initiative_meta_ai_object"
                indexName="idx_initiative_meta_ai_object_initiative_active">
            <column name="initiative_id"/>
            <column name="is_active"/>
        </createIndex>

        <rollback>
            <dropTable tableName="initiative_meta_ai_object"/>
        </rollback>

    </changeSet>

</databaseChangeLog>



<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd">

    <changeSet id="create-initiative-ai-object-history" author="PolitovAY">

        <createTable tableName="initiative_ai_object_history"
                     remarks="История изменений AI-объектов инициатив">

            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true"
                             primaryKeyName="pk_initiative_ai_object_history"
                             nullable="false"/>
            </column>

            <column name="ai_agent_id" type="BIGINT"
                    remarks="FK на ai_agent.id">
                <constraints nullable="false"/>
            </column>

            <column name="file_upload_id" type="BIGINT"
                    remarks="ID файловой загрузки"/>

            <column name="changed_objects_count" type="INTEGER"
                    remarks="Количество измененных объектов">
                <constraints nullable="false"/>
            </column>

            <column name="event_type" type="VARCHAR(32)"
                    remarks="Тип события">
                <constraints nullable="false"/>
            </column>

            <column name="created_by" type="VARCHAR"
                    remarks="ID пользователя"/>

            <column name="created_by_name" type="VARCHAR(500)"
                    remarks="Отображаемое имя пользователя"/>

            <column name="created_at" type="TIMESTAMP"
                    defaultValueComputed="CURRENT_TIMESTAMP"
                    remarks="Дата события">
                <constraints nullable="false"/>
            </column>

        </createTable>

        <addForeignKeyConstraint
                baseTableName="initiative_ai_object_history"
                baseColumnNames="ai_agent_id"
                constraintName="fk_initiative_ai_object_history_ai_agent"
                referencedTableName="ai_agent"
                referencedColumnNames="id"/>

        <createIndex
                tableName="initiative_ai_object_history"
                indexName="idx_initiative_ai_object_history_agent_created">
            <column name="ai_agent_id"/>
            <column name="created_at"/>
            <column name="id"/>
        </createIndex>

        <createIndex
                tableName="initiative_ai_object_history"
                indexName="idx_initiative_ai_object_history_file_upload">
            <column name="file_upload_id"/>
        </createIndex>

        <rollback>
            <dropTable tableName="initiative_ai_object_history"/>
        </rollback>

    </changeSet>

</databaseChangeLog>

```
