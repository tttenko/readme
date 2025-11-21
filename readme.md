```java
if (StringUtils.isNotBlank(properties.getAttributeIdForTbDzo())
        && properties.getSlugValueForTerBank().equals(dictionaryName)) {
        builder.addBoolEqualsByAttrId(
            properties.getAttributeIdForTbDzo(),   // UUID атрибута DZO у ТБ
            properties.isDzoAttributeValue());     // обычно false
    }
```
