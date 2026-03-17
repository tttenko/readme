```java

<?xml version="1.1" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.8.xsd">

    <changeSet id="12022026/create_sts_data" author="koptenko">
        <comment>Create sts_data table for special transport data</comment>
        <sql>
            create table if not exists sts_data (
                uuid uuid not null default gen_random_uuid(),
                contract_uuid uuid not null,
                tb_code varchar(4) not null,
                vehicle_number varchar(15) not null,
                vehicle_brand varchar(35) not null,
                comment varchar(128),
                status_code varchar(2) not null,
                created_by varchar(100) not null,
                updated_by varchar(100) not null,
                created_at timestamp with time zone not null,
                updated_at timestamp with time zone not null,
                is_deleted boolean not null default false,

                constraint pk_sts_data primary key (uuid)
            );

            create index if not exists idx_sts_data_contract_uuid
                on sts_data (contract_uuid);

            create index if not exists idx_sts_data_status_code
                on sts_data (status_code);

            create index if not exists idx_sts_data_tb_code
                on sts_data (tb_code);

            create index if not exists idx_sts_data_created_at
                on sts_data (created_at);
        </sql>
    </changeSet>

</databaseChangeLog>


```
