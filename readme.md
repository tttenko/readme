```java
<?xml version="1.1" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                      http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.6.xsd">

  <changeSet id="12022026/create_fx_rate" author="tttenko">
    <comment>Create fx_rate table for currency/metals rates</comment>
    <sql>
      create table if not exists fx_rate (
        id bigserial not null,

        rquid uuid not null,
        rqtm  timestamp not null,

        fxratesubtype varchar(8)  not null,
        code1         varchar(16) not null,
        isonum1       varchar(16),

        code2         varchar(16) not null,
        isonum2       varchar(16),

        use_date date not null,

        lot_size numeric(19,8),
        value    numeric(19,8),

        is_public varchar(16),

        constraint pk_fx_rate primary key (id),
        constraint ux_fx_rate_key unique (fxratesubtype, code1, code2, use_date)
      );

      create index if not exists ix_fx_rate_lookup
        on fx_rate (code1, code2, use_date desc);

      create index if not exists ix_fx_rate_subtype
        on fx_rate (fxratesubtype);
    </sql>
  </changeSet>

</databaseChangeLog>
```
