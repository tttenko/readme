```java
private fun updateAgentDivisionOrBlock(
    entity: AIAgentEntity,
    headers: Map<String, Int>,
    row: Row,
    divisionsDict: List<DivisionsDto>,
    availableDivisions: MutableList<DivisionEntity>,
    availableBlocks: MutableList<BlockEntity>,
    fileId: Long
) {
    val divisionCellValue = headers[DIVISION]?.let { headerIndex ->
        getValueOrNull(row.getCell(headerIndex))
            ?.takeUnless(String::isBlank)
    }

    if (divisionCellValue == null) {
        val blockCellValue = headers[BLOCK]?.let { headerIndex ->
            getValueOrNull(row.getCell(headerIndex))
                ?.takeUnless(String::isBlank)
        }

        val block = availableBlocks.find {
            it.shortName.equals(
                blockCellValue?.trim(),
                ignoreCase = true
            )
        }

        if (block == null) {
            throw AiFileUploadException(
                AI_UPLOAD_EMPTY_TRIBE_NAME,
                messageProvider[
                    AI_UPLOAD_EMPTY_TRIBE_NAME,
                    blockCellValue,
                    row.rowNum.plus(1)
                ],
                fileId = fileId
            )
        }

        // Агент относится напрямую к блоку, без трайба
        entity.division = null
        entity.block = block

        return
    }

    val division = divisionsDict
        .firstOrNull {
            it.shortName.equals(
                divisionCellValue.trim(),
                ignoreCase = true
            )
        }
        ?.let { divisionFromDict ->
            availableDivisions.find {
                it.code.equals(
                    divisionFromDict.code.trim(),
                    ignoreCase = true
                )
            }
        }

    if (division == null) {
        throw AiFileUploadException(
            AI_UPLOAD_EMPTY_TRIBE_NAME,
            messageProvider[
                AI_UPLOAD_EMPTY_TRIBE_NAME,
                row.rowNum.plus(1)
            ],
            fileId = fileId
        )
    }

    // При наличии трайба сохраняем и сам трайб, и связанный с ним блок
    entity.division = division
    entity.block = division.block
}
  ```
