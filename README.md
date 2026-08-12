```java

private fun updateInvolvedResource(
    row: Row,
    headerRow: Row?,
    resourcesStartIndex: Int,
    dictionaries: AgentImportDictionaries,
    entity: AIAgentEntity,
    fileId: Long
) {
    val involvedResourceDuration = measureTimeMillis {
        if (resourcesStartIndex == -1 || headerRow == null) {
            return@measureTimeMillis
        }

        val agentId = requireNotNull(entity.id) {
            "Agent id must be initialized before involved resources update"
        }

        val resourcesEndIndex = resourcesStartIndex + 3

        /*
         * Сначала собираем желаемое состояние из Excel.
         *
         * Важно: здесь НЕ создаем InvolvedResourceEntity,
         * потому что entity с таким composite id уже может быть managed
         * текущим Hibernate Session.
         */
        val desiredResources =
            linkedMapOf<Pair<String?, String?>, BigDecimal>()

        row
            .filter { it.columnIndex in resourcesStartIndex..resourcesEndIndex }
            .forEach { resourceCell ->

                val resourceName = headerRow
                    .getCell(resourceCell.columnIndex)
                    ?.stringCellValue
                    ?.trim()
                    ?.takeIf { it.isNotBlank() }
                    ?: return@forEach

                val rawValue = getValueOrNull(
                    row.getCell(resourceCell.columnIndex)
                )
                    ?.takeIf { it.isNotBlank() }
                    ?: return@forEach

                val resourceRef = dictionaries.resourcesByName[resourceName]
                    ?: throw AiFileUploadException(
                        AI_UPLOAD_UNKNOWN_METRIC_NAME,
                        MessageFormat.format(
                            messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                            resourceCell.columnIndex.plus(1),
                            row.rowNum.plus(1)
                        ),
                        fileId = fileId
                    )

                val numericValue = try {
                    rawValue
                        .replace(',', '.')
                        .toBigDecimal()
                } catch (e: NumberFormatException) {
                    throw AiFileUploadException(
                        AI_UPLOAD_UNKNOWN_METRIC_NAME,
                        MessageFormat.format(
                            messageProvider[AI_UPLOAD_UNKNOWN_METRIC_NAME],
                            resourceCell.columnIndex.plus(1),
                            row.rowNum.plus(1)
                        ),
                        fileId = fileId
                    )
                }

                val key =
                    resourceRef.source to resourceRef.type

                desiredResources[key] = numericValue
            }

        /*
         * Сущности из этой коллекции уже managed.
         *
         * Индексируем их по той части composite id,
         * которая уникальна внутри одного агента.
         */
        val existingByKey = entity.involvedResource
            .associateBy { resource ->
                resource.id.source to resource.id.type
            }

        /*
         * Удаляем только те ресурсы, которых больше нет в Excel.
         *
         * orphanRemoval=true удалит соответствующую строку из БД
         * при flush.
         */
        val iterator = entity.involvedResource.iterator()

        while (iterator.hasNext()) {
            val existing = iterator.next()

            val key =
                existing.id.source to existing.id.type

            if (key !in desiredResources) {
                iterator.remove()
            }
        }

        val now = LocalDateTime.now()

        /*
         * Existing entity -> изменяем тот же managed object.
         *
         * New entity -> только тогда создаем новый объект.
         */
        desiredResources.forEach { (key, value) ->

            val existing = existingByKey[key]

            if (existing != null) {
                existing.value = value
                existing.updated = now
                existing.aiAgent = entity
            } else {
                entity.involvedResource.add(
                    InvolvedResourceEntity().apply {
                        id = InvolvedResourceEmbeddedId(
                            aiAgentId = agentId,
                            source = key.first,
                            type = key.second
                        )
                        this.value = value
                        aiAgent = entity
                        updated = now
                    }
                )
            }
        }
    }

    log.debug(
        "Operation involved Resource took $involvedResourceDuration ms"
    )
}

private fun updateImplementedPlatform(
    platformsStartIndex: Int,
    platformsEndIndex: Int,
    headerRow: Row?,
    row: Row,
    entity: AIAgentEntity,
    dictionaries: AgentImportDictionaries
) {
    val implementedPlatformDuration = measureTimeMillis {
        if (
            platformsStartIndex == -1 ||
            platformsEndIndex == -1 ||
            headerRow == null
        ) {
            return@measureTimeMillis
        }

        val agentId = requireNotNull(entity.id) {
            "Agent id must be initialized before implemented platforms update"
        }

        /*
         * platformId -> released
         *
         * Сначала только собираем желаемое состояние.
         * JPA entities здесь не создаем.
         */
        val desiredPlatforms = linkedMapOf<Long, Boolean>()

        row
            .filter {
                it.columnIndex in
                    platformsStartIndex..platformsEndIndex
            }
            .forEach { platformCell ->

                val cellHeader = headerRow
                    .getCell(platformCell.columnIndex)
                    ?.stringCellValue
                    ?.trim()
                    ?.takeIf { it.isNotBlank() }
                    ?: return@forEach

                val rawValue = getValueOrNull(
                    row.getCell(platformCell.columnIndex)
                )
                    ?.takeIf { it.isNotBlank() }
                    ?: return@forEach

                val platformRef =
                    dictionaries.platformsByName[cellHeader]
                        ?: return@forEach

                val released = when {
                    rawValue.equals(
                        "Внедрен",
                        ignoreCase = true
                    ) -> true

                    rawValue.equals(
                        "В разработке",
                        ignoreCase = true
                    ) -> false

                    else -> return@forEach
                }

                desiredPlatforms[platformRef.id] = released
            }

        /*
         * Существующие элементы коллекции уже managed Hibernate.
         */
        val existingByPlatformId = entity.platforms
            .associateBy { implementedPlatform ->
                implementedPlatform.primaryKey.platformId
            }

        /*
         * Платформы, которых больше нет в Excel, удаляем.
         *
         * Для platforms установлен orphanRemoval=true,
         * поэтому Hibernate удалит соответствующие строки.
         */
        val iterator = entity.platforms.iterator()

        while (iterator.hasNext()) {
            val existing = iterator.next()

            if (
                existing.primaryKey.platformId
                    !in desiredPlatforms
            ) {
                iterator.remove()
            }
        }

        /*
         * Для существующего composite key используем существующую
         * managed entity.
         */
        desiredPlatforms.forEach { (platformId, released) ->

            val existing =
                existingByPlatformId[platformId]

            if (existing != null) {
                existing.released = released
                existing.aiAgent = entity
            } else {
                entity.platforms.add(
                    ImplementedPlatformEntity().apply {
                        primaryKey =
                            AIAgentPlatformPK().apply {
                                aiAgentId = agentId
                                this.platformId = platformId
                            }

                        platform = managedReference(
                            PlatformEntity::class.java,
                            platformId
                        )

                        aiAgent = entity
                        this.released = released
                    }
                )
            }
        }
    }

    log.debug(
        "Operation implemented platforms took $implementedPlatformDuration ms"
    )
}
```
