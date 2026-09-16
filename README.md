```java

/**
     * Получает пользователей по набору идентификаторов одним batch-вызовом.
     *
     * При недоступности prm-auth возвращает пустую map.
     * Вызывающий код в этом случае может использовать userId как fallback.
     */
    fun getUsersByIds(userIds: Set<Long>): Map<Long, UserAccountDto> {
        if (userIds.isEmpty()) {
            return emptyMap()
        }

        return try {
            prmAuthFeignClient.getUsers(userIds, null).body.orEmpty().associateBy { it.id }
        } catch (exception: RetryableException) {
            log.warn(
                "Не удалось получить пользователей из prm-auth: сервис недоступен, userIds={}",
                userIds,
                exception
            )
            emptyMap()
        } catch (exception: FeignException) {
            log.warn(
                "Ошибка prm-auth при получении пользователей, status={}, userIds={}",
                exception.status(),
                userIds,
                exception
            )
            emptyMap()
        }
    }
```
