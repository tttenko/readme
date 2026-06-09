```java

if (request.containsKey("platforms")) {

    val platforms = handlePlatforms(request)

    if (platforms.isNullOrEmpty()) {
        implementedPlatformRepository.deleteAllByAiAgentId(
            aiAgentId = agent.id
        )

        agent.platforms.clear()
    } else {
        val platformEntitiesByCode =
            platforms.associate { platformDto ->

                val platformCode = platformDto.code

                val platformEntity =
                    platformRepository.findFirstByCode(
                        code = platformCode
                    ) ?: throw AiBadRequestException(
                        errorCode = PLATFORM_WITH_CODE_NOT_FOUND,
                        message = MessageFormat.format(
                            messageProvider[
                                PLATFORM_WITH_CODE_NOT_FOUND
                            ],
                            platformCode,
                        ),
                        operationDetails = operationDetails,
                    )

                platformCode to platformEntity
            }

        /*
         * Старые связи удаляются только после того,
         * как все коды платформ успешно провалидированы.
         */
        implementedPlatformRepository.deleteAllByAiAgentId(
            aiAgentId = agent.id
        )

        agent.platforms.clear()

        platforms.forEach { platformDto ->

            val platformEntity =
                requireNotNull(
                    platformEntitiesByCode[platformDto.code]
                )

            agent.platforms.add(
                ImplementedPlatformEntity().apply {
                    primaryKey = AIAgentPlatformPK().apply {
                        aiAgentId = agent.id
                        platformId = platformEntity.id
                    }

                    platform = platformEntity
                    aiAgent = agent
                    released = platformDto.released
                }
            )
        }
    }
}
```
