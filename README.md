```java
private fun buildMessage(history: InitiativeAiObjectHistoryEntity): String {
    val count = history.changedObjectsCount
    val objectWord = if (count % 10 == 1 && count % 100 != 11) {
        "объекту"
    } else {
        "объектам"
    }

    return when (history.eventType) {
        InitiativeAiObjectEventType.FILE_IMPORT ->
            "Получены новые сведения по $count $objectWord инициативы"
    }
}
```
