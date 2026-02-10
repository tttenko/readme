```java
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
      http://www.liquibase.org/xml/ns/dbchangelog
      http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

  <!--
    Таблица для хранения курсов валют/цен металлов из Kafka XML.

    Ключ идемпотентности (upsert): (fxratesubtype, code1, code2, use_date)

    Оптимизация под REST:
      - запрос "найти запись с максимальной use_date, где use_date <= :date и code1=:code1 и code2=:code2"
      - индекс (code1, code2, use_date DESC)
  -->

  <changeSet id="001-create-fx-rate" author="tttenko">
    <preConditions onFail="MARK_RAN">
      <not>
        <tableExists tableName="fx_rate"/>
      </not>
    </preConditions>

    <createTable tableName="fx_rate">
      <column name="id" type="BIGSERIAL">
        <constraints primaryKey="true" nullable="false" primaryKeyName="pk_fx_rate"/>
      </column>

      <column name="rquid" type="UUID">
        <constraints nullable="false"/>
      </column>

      <column name="rqtm" type="TIMESTAMP">
        <constraints nullable="false"/>
      </column>

      <column name="fxratesubtype" type="VARCHAR(8)">
        <constraints nullable="false"/>
      </column>

      <column name="code1" type="VARCHAR(16)">
        <constraints nullable="false"/>
      </column>

      <column name="isonum1" type="VARCHAR(16)"/>

      <column name="code2" type="VARCHAR(16)">
        <constraints nullable="false"/>
      </column>

      <column name="isonum2" type="VARCHAR(16)"/>

      <!-- ✅ ключевой момент: храним именно ДАТУ, без времени -->
      <column name="use_date" type="DATE">
        <constraints nullable="false"/>
      </column>

      <column name="lot_size" type="NUMERIC(19,8)"/>
      <column name="value" type="NUMERIC(19,8)"/>

      <!-- опционально, но полезно сохранять -->
      <column name="is_public" type="VARCHAR(16)"/>
    </createTable>
  </changeSet>

  <!-- Уникальность: чтобы consumer был идемпотентным при повторах Kafka -->
  <changeSet id="002-ux-fx-rate-key" author="tttenko">
    <preConditions onFail="MARK_RAN">
      <not>
        <uniqueConstraintExists tableName="fx_rate" constraintName="ux_fx_rate_key"/>
      </not>
    </preConditions>

    <addUniqueConstraint
        tableName="fx_rate"
        columnNames="fxratesubtype,code1,code2,use_date"
        constraintName="ux_fx_rate_key"/>
  </changeSet>

  <!-- Индекс под выборку в REST: code1+code2 и свежайшая дата первой -->
  <changeSet id="003-ix-fx-rate-lookup" author="tttenko">
    <preConditions onFail="MARK_RAN">
      <not>
        <indexExists tableName="fx_rate" indexName="ix_fx_rate_lookup"/>
      </not>
    </preConditions>

    <!-- Для DESC в индексе проще и надёжнее raw SQL -->
    <sql>
      CREATE INDEX IF NOT EXISTS ix_fx_rate_lookup
      ON fx_rate (code1, code2, use_date DESC);
    </sql>
  </changeSet>

  <!-- (Опционально) индекс по subtype, если часто фильтруете по FXCB/PMCB -->
  <changeSet id="004-ix-fx-rate-subtype" author="tttenko">
    <preConditions onFail="MARK_RAN">
      <not>
        <indexExists tableName="fx_rate" indexName="ix_fx_rate_subtype"/>
      </not>
    </preConditions>

    <createIndex indexName="ix_fx_rate_subtype" tableName="fx_rate">
      <column name="fxratesubtype"/>
    </createIndex>
  </changeSet>

</databaseChangeLog>
```
