```java
<?xml version="1.1" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.6.xsd">

    <changeSet id="CONTROL-VEHICLE/create_outbox_message/10" author="OpenAI">
        <preConditions onFail="HALT" onError="HALT">
            <not>
                <tableExists tableName="outbox_message"/>
            </not>
        </preConditions>
        <comment>Create table OUTBOX_MESSAGE</comment>
        <sql>
            CREATE TABLE OUTBOX_MESSAGE (
                UUID            UUID NOT NULL,
                EVENT_TYPE      VARCHAR(255) NOT NULL,
                AGGREGATE_ID    VARCHAR(255) NOT NULL,
                PAYLOAD         JSONB,
                CREATED_AT      TIMESTAMP WITH TIME ZONE NOT NULL,
                CREATED_BY      UUID NOT NULL,
                PUBLISHED_AT    TIMESTAMP WITH TIME ZONE,
                NEXT_ATTEMPT_AT TIMESTAMP WITH TIME ZONE NOT NULL,
                PICKED_AT       TIMESTAMP WITH TIME ZONE,
                STATUS          VARCHAR(50) NOT NULL,

                CONSTRAINT PK_OUTBOX_MESSAGE PRIMARY KEY (UUID)
            );
        </sql>
    </changeSet>

    <changeSet id="CONTROL-VEHICLE/create_outbox_message/20" author="OpenAI">
        <preConditions onFail="HALT" onError="HALT">
            <not>
                <tableExists tableName="outbox_message_error"/>
            </not>
        </preConditions>
        <comment>Create table OUTBOX_MESSAGE_ERROR</comment>
        <sql>
            CREATE TABLE OUTBOX_MESSAGE_ERROR (
                UUID                UUID NOT NULL,
                ATTEMPTED_AT        TIMESTAMP WITH TIME ZONE NOT NULL,
                ERROR_MESSAGE       TEXT,
                OUTBOX_MESSAGE_UUID UUID NOT NULL,

                CONSTRAINT PK_OUTBOX_MESSAGE_ERROR PRIMARY KEY (UUID),
                CONSTRAINT FK_OUTBOX_MESSAGE_ERROR_OUTBOX_MESSAGE
                    FOREIGN KEY (OUTBOX_MESSAGE_UUID)
                    REFERENCES OUTBOX_MESSAGE(UUID)
                    ON DELETE CASCADE
            );
        </sql>
    </changeSet>

    <changeSet id="CONTROL-VEHICLE/create_outbox_message/30" author="OpenAI">
        <preConditions onFail="MARK_RAN" onError="HALT">
            <not>
                <indexExists indexName="idx_outbox_message_pending_next_attempt"/>
            </not>
        </preConditions>
        <comment>Index for searching pending outbox messages</comment>
        <sql>
            CREATE INDEX idx_outbox_message_pending_next_attempt
                ON OUTBOX_MESSAGE (STATUS, NEXT_ATTEMPT_AT);
        </sql>
    </changeSet>

    <changeSet id="CONTROL-VEHICLE/create_outbox_message/40" author="OpenAI">
        <preConditions onFail="MARK_RAN" onError="HALT">
            <not>
                <indexExists indexName="idx_outbox_message_in_progress_picked_next_attempt"/>
            </not>
        </preConditions>
        <comment>Index for searching in-progress outbox messages</comment>
        <sql>
            CREATE INDEX idx_outbox_message_in_progress_picked_next_attempt
                ON OUTBOX_MESSAGE (STATUS, PICKED_AT, NEXT_ATTEMPT_AT);
        </sql>
    </changeSet>

    <changeSet id="CONTROL-VEHICLE/create_outbox_message/50" author="OpenAI">
        <preConditions onFail="MARK_RAN" onError="HALT">
            <not>
                <indexExists indexName="idx_outbox_message_event_aggregate_created"/>
            </not>
        </preConditions>
        <comment>Index for checking newer messages by event type and aggregate</comment>
        <sql>
            CREATE INDEX idx_outbox_message_event_aggregate_created
                ON OUTBOX_MESSAGE (EVENT_TYPE, AGGREGATE_ID, CREATED_AT);
        </sql>
    </changeSet>

</databaseChangeLog>
```
