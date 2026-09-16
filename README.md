```java

fun getUsersByIds(userIds: Set<Long>): Map<Long, UserAccountDto> {
    if (userIds.isEmpty()) {
        return emptyMap()
    }

    return try {
        prmAuthFeignClient.getUsers(userIds, null).body.orEmpty().associateBy { it.id }
    } catch (exception: FeignException) {
        log.warn(
            "Не удалось получить пользователей из prm-auth, status={}, userIds={}",
            exception.status(),
            userIds,
            exception
        )
        emptyMap()
    }
}
```
