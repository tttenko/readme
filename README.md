```java
<changeSet id="Alter updated_by type in metrics_directory" author="KoptenkoMV">

    <sql>
        ALTER TABLE metrics_directory
            ALTER COLUMN updated_by TYPE BIGINT
            USING updated_by::BIGINT;
    </sql>

    <rollback>
        <sql>
            ALTER TABLE metrics_directory
                ALTER COLUMN updated_by TYPE VARCHAR(255)
                USING updated_by::VARCHAR;
        </sql>
    </rollback>

</changeSet>
```
