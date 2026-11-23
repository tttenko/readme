```java


/**
 * Результат разбора XLS/XLSX-файла.
 *
 * @property rows корректно разобранные строки
 * @property errors ошибки валидации отдельных строк
 */
data class MetaAiObjectsParsedFile(
    val rows: List<MetaAiObjectsExcelRow>,
    val errors: List<String>
)

```
