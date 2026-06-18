```java
fun UserAccountDto.toFullName(): String {
    return listOf(
        lastName,
        firstName,
        patronymic,
    )
        .mapNotNull { value ->
            value?.trim()
        }
        .filter { value ->
            value.isNotBlank()
        }
        .joinToString(separator = " ")
}
```
