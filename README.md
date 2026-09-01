```java
private fun <T> findByLabel(
    values: List<String>?,
    entitiesByLabel: Map<String, T>,
): T? {
    return values
        .orEmpty()
        .asSequence()
        .flatMap { value ->
            entitiesByLabel
                .asSequence()
                .filter { (label, _) ->
                    label.isNotBlank() &&
                        value.contains(label, ignoreCase = true)
                }
        }
        .maxByOrNull { (label, _) -> label.length }
        ?.value
}

А дальше:

private fun findDivision(
    values: List<String>?,
    divisionsByLabel: Map<String, DivisionEntity>,
): DivisionEntity? =
    findByLabel(
        values = values,
        entitiesByLabel = divisionsByLabel,
    )

private fun findBlock(
    values: List<String>?,
    blocksByLabel: Map<String, BlockEntity>,
): BlockEntity? =
    findByLabel(
        values = values,
        entitiesByLabel = blocksByLabel,
    )
```
