```java

private fun findDivision(
    values: List<String>?,
    divisionsByLabel: Map<String, DivisionEntity>,
): DivisionEntity? {
    return values
        .orEmpty()
        .flatMap { value ->
            divisionsByLabel
                .filterKeys { label ->
                    label.isNotBlank() &&
                        value.contains(label, ignoreCase = true)
                }
                .entries
        }
        .maxByOrNull { entry -> entry.key.length }
        ?.value
}

private fun findBlock(
    values: List<String>?,
    blocksByLabel: Map<String, BlockEntity>,
): BlockEntity? {
    return values
        .orEmpty()
        .flatMap { value ->
            blocksByLabel
                .filterKeys { label ->
                    label.isNotBlank() &&
                        value.contains(label, ignoreCase = true)
                }
                .entries
        }
        .maxByOrNull { entry -> entry.key.length }
        ?.value
}
```
