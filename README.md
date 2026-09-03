```java
ExcelColumnDescription(
    "Архив",
    { param ->
        when (param.first.isDisable) {
            true -> param.third.setCellValue("Да")
            false -> param.third.setCellValue("Нет")
            null -> Unit
        }
    }
),
```
