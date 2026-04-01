```java
/**
     * Обрабатывает одно сообщение по его идентификатору.
     * Вызывается при получении события AFTER_COMMIT.
     *
     * @param outboxMessageId идентификатор сообщения
     */
    public void processOutboxMessage(UUID outboxMessageId) {
        OutboxMessage outboxMessage =
                outboxMessagePersistenceService.findAndLockPendingMessageById(outboxMessageId);

        if (outboxMessage == null) {
            log.warn("Outbox message was not found or already locked/processed. uuid={}", outboxMessageId);
            return;
        }

        processSingleMessageSafely(outboxMessage);
    }

    private void processSingleMessageSafely(OutboxMessage message) {
        try {
            dispatcher.dispatch(message);
        } catch (Exception ex) {
            log.error("Unexpected outbox dispatch error. UUID '{}', event '{}', aggregate id '{}'",
                    message.getUuid(), message.getEventType(), message.getAggregateId(), ex);

            try {
                message.setFailed(safeErrorMessage(ex));
                outboxMessagePersistenceService.save(message);
            } catch (Exception saveEx) {
                log.error("Failed to mark outbox message as FAILED. UUID '{}'",
                        message.getUuid(), saveEx);
            }
        }
    }

    private String safeErrorMessage(Exception error) {
        return error.getMessage() == null ? error.getClass().getSimpleName() : error.getMessage();
    }
```
