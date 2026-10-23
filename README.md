```java
private fun parseFinancialEffect(cell: PreparedCell?): BigDecimal? {
    val value = getValueOrNull(cell) ?: return null

    return value
        .replace(Regex("[\\s\\u00A0\\u202F]"), "")
        .replace(',', '.')
        .toDouble()
        .toBigDecimal()
}


```
