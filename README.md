```java

<column name="crossgoal" type="VARCHAR(255)"
        remarks="Бизнес-идентификатор инициативы CROSSGOAL">
    <constraints nullable="false"/>
</column>

<createIndex
        tableName="initiative_meta_ai_object"
        indexName="idx_initiative_meta_ai_object_crossgoal">
    <column name="crossgoal"/>
</createIndex>
```
