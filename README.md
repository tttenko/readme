```java
<sql>
        CREATE SEQUENCE IF NOT EXISTS enabler_id_seq;

        ALTER TABLE enabler
            ALTER COLUMN id SET DEFAULT nextval('enabler_id_seq');

        SELECT setval(
            'enabler_id_seq',
            COALESCE((SELECT MAX(id) FROM enabler), 0) + 1,
            false
        );
    </sql>

    <rollback>
        <sql>
            ALTER TABLE enabler
                ALTER COLUMN id DROP DEFAULT;

            DROP SEQUENCE IF EXISTS enabler_id_seq;
        </sql>
    </rollback>

```
