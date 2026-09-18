```java
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://www.liquibase.org/xml/ns/dbchangelog
            http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <changeSet id="Add metric applicability request decision email templates"
               author="KoptenkoMV">

        <insert tableName="email_template">
            <column name="id"
                    value="unlinkMetricFromInitiativeRequestApprove"/>

            <column name="subject">
                <value><![CDATA[${agentName}: метрика больше не актуальна]]></value>
            </column>

            <column name="template">
                <value><![CDATA[
Здравствуйте!

Метрика «${metric}» признана неактуальной для инициативы «${agentName}».
Открыть инициативу: ${link}

Поддержка Пульта
${supportEmail}
                ]]></value>
            </column>
        </insert>

        <insert tableName="email_template">
            <column name="id"
                    value="unlinkMetricFromInitiativeRequestReject"/>

            <column name="subject">
                <value><![CDATA[${agentName}: метрика признана актуальной]]></value>
            </column>

            <column name="template">
                <value><![CDATA[
Здравствуйте!

Метрика «${metric}» остаётся актуальной для инициативы «${agentName}».
Открыть инициативу: ${link}

Поддержка Пульта
${supportEmail}
                ]]></value>
            </column>
        </insert>

        <insert tableName="email_template">
            <column name="id"
                    value="unlinkMetricFromInitiativeRequestCancelDecision"/>

            <column name="subject">
                <value><![CDATA[${agentName}: актуальность метрики изменилась]]></value>
            </column>

            <column name="template">
                <value><![CDATA[
Здравствуйте!

Сотрудник офиса AI-трансформации отменил решение по актуальности метрики «${metric}» для инициативы «${agentName}».
Открыть инициативу: ${link}

Поддержка Пульта
${supportEmail}
                ]]></value>
            </column>
        </insert>

    </changeSet>

</databaseChangeLog>
```
