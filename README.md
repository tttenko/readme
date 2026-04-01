```java
@SuppressWarnings({"java:S2245", "java:S2140"})
    public void execute(OutboxMessage message) {
        try {
            if (message.getEventType().isNegligible()) {
                boolean isObsolete = outboxMessagePersistenceService.existsNewerMessage(
                        message.getEventType(),
                        message.getAggregateId(),
                        message.getCreatedAt()
                );

                if (isObsolete) {
                    log.warn("Skipping obsolete outbox message: event '{}', aggregate ID '{}'",
                            message.getEventType(), message.getAggregateId());

                    message.setCanceled();
                    outboxMessagePersistenceService.save(message);
                    return;
                }
            }

            log.info("Starting processing outbox message: UUID '{}', event '{}', aggregate id '{}'",
                    message.getUuid(), message.getEventType(), message.getAggregateId());

            action(message);

            message.setDone();
            outboxMessagePersistenceService.save(message);

            log.info("Successfully processed outbox message: UUID '{}', event '{}', aggregate id '{}'",
                    message.getUuid(), message.getEventType(), message.getAggregateId());

        } catch (Exception error) {

            if (message.getErrors().size() >= maximumOfRetries - 1
                    || !message.getEventType().isRetryable()) {

                log.error("Failed to process outbox message: UUID '{}', event '{}', aggregate id '{}'",
                        message.getUuid(), message.getEventType(), message.getAggregateId(), error);

                message.setFailed(safeErrorMessage(error));

            } else {
                long baseDelay = initialRetryDelay * (long) Math.pow(2, message.getErrors().size());
                long jitterDelay = (long) (baseDelay * (0.8 + 0.4 * Math.random()));

                message.retryExecution(
                        safeErrorMessage(error),
                        Math.min(jitterDelay, maximumRetryDelay)
                );

                log.error("Outbox message processing failed. The message will be automatically retried: "
                                + "UUID '{}', event '{}', aggregate id '{}', next attempt at '{}'",
                        message.getUuid(),
                        message.getEventType(),
                        message.getAggregateId(),
                        message.getNextAttemptAt(),
                        error);
            }

            outboxMessagePersistenceService.save(message);
        }
    }

    public abstract OutboxMessageEventType getEventType();

    protected abstract void action(OutboxMessage message) throws JsonProcessingException;

    private String safeErrorMessage(Exception error) {
        return error.getMessage() == null ? error.getClass().getSimpleName() : error.getMessage();
    }
}
```
