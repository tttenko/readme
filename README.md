```java
public void sendHistory(HistoryNewDto dto) {
    String key = dto.getEntityUuid().toString();

    try {
        historyNewTemplate.send(historyTopic, key, dto).get();

        log.info("The status history has been sent to Kafka. topic={}, entityUuid={}, status={}",
                historyTopic, dto.getEntityUuid(), dto.getStatus());

    } catch (InterruptedException ex) {
        Thread.currentThread().interrupt();
        throw new IllegalStateException(
                "Failed to send tracker history event to Kafka. entityUuid=" + dto.getEntityUuid(), ex);

    } catch (ExecutionException ex) {
        throw new IllegalStateException(
                "Failed to send tracker history event to Kafka. entityUuid=" + dto.getEntityUuid(), ex);
    }
}

public void sendAdditional(PlannedDateDto dto) {
    String key = dto.getEntityUuid().toString();

    try {
        plannedDateTemplate.send(additionalTopic, key, dto).get();

        log.info("Planned date has been sent to Kafka. topic={}, entityUuid={}",
                additionalTopic, dto.getEntityUuid());

    } catch (InterruptedException ex) {
        Thread.currentThread().interrupt();
        throw new IllegalStateException(
                "Failed to send additional event to Kafka. entityUuid=" + dto.getEntityUuid(), ex);

    } catch (ExecutionException ex) {
        throw new IllegalStateException(
                "Failed to send additional event to Kafka. entityUuid=" + dto.getEntityUuid(), ex);
    }
}

                
```
