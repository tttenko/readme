```java

<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://www.liquibase.org/xml/ns/dbchangelog
            http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.5.xsd">

    <changeSet id="Update pending metric applicability request unique index" author="KoptenkoMV">

        <!--
            PENDING-заявка считается активной только пока она видима Офису.

            После отмены координатором заявка остаётся PENDING для сохранения
            истории, но переводится в is_visible_in_office=false.

            Поэтому ограничение "не более одной PENDING-заявки"
            распространяется только на видимые заявки.
        -->
        <sql>
            DROP INDEX IF EXISTS uq_metric_applicability_request_pending_assignment;

            CREATE UNIQUE INDEX uq_metric_applicability_request_pending_assignment
            ON metric_applicability_request (initiative_metric_assignment_id)
            WHERE status = 'PENDING'
              AND is_visible_in_office = true;
        </sql>

        <rollback>
            <sql>
                DROP INDEX IF EXISTS uq_metric_applicability_request_pending_assignment;

                CREATE UNIQUE INDEX uq_metric_applicability_request_pending_assignment
                ON metric_applicability_request (initiative_metric_assignment_id)
                WHERE status = 'PENDING';
            </sql>
        </rollback>

    </changeSet>

</databaseChangeLog>
```
