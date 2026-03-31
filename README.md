```java
@ExtendWith(MockitoExtension.class)
class OutboxMessageActionTest {

    @Mock
    private OutboxMessagePersistenceService outboxMessagePersistenceService;

    @AfterEach
    void tearDown() {
        setEventTypeFlags(OutboxMessageEventType.CREATE_EVENT, false, true);
    }

    @Test
    void execute_shouldMarkDone_whenActionSucceeded() {
        UUID messageId = UUID.randomUUID();

        OutboxMessage message = newMessage(
                messageId,
                OutboxMessageEventType.CREATE_EVENT,
                OutboxMessageStatus.IN_PROGRESS
        );

        TestOutboxMessageAction action =
                new TestOutboxMessageAction(outboxMessagePersistenceService, null);

        setRetryProperties(action);

        when(outboxMessagePersistenceService.findUnpublishedMessageByIdWithErrors(messageId))
                .thenReturn(message);
        when(outboxMessagePersistenceService.save(any(OutboxMessage.class)))
                .thenAnswer(invocation -> invocation.getArgument(0));

        action.execute(messageId);

        assertTrue(action.wasCalled());

        verify(outboxMessagePersistenceService).findUnpublishedMessageByIdWithErrors(messageId);
        verify(outboxMessagePersistenceService).save(argThat(saved ->
                saved.getStatus() == OutboxMessageStatus.DONE
                        && saved.getPublishedAt() != null
        ));
        verifyNoMoreInteractions(outboxMessagePersistenceService);
    }

    @Test
    void execute_shouldCancelMessage_whenEventIsNegligibleAndObsolete() {
        UUID messageId = UUID.randomUUID();

        setEventTypeFlags(OutboxMessageEventType.CREATE_EVENT, true, true);

        OutboxMessage message = newMessage(
                messageId,
                OutboxMessageEventType.CREATE_EVENT,
                OutboxMessageStatus.PENDING
        );

        TestOutboxMessageAction action =
                new TestOutboxMessageAction(outboxMessagePersistenceService, null);

        setRetryProperties(action);

        when(outboxMessagePersistenceService.findUnpublishedMessageByIdWithErrors(messageId))
                .thenReturn(message);
        when(outboxMessagePersistenceService.existsNewerMessage(
                message.getEventType(),
                message.getAggregateId(),
                message.getCreatedAt()
        )).thenReturn(true);
        when(outboxMessagePersistenceService.save(any(OutboxMessage.class)))
                .thenAnswer(invocation -> invocation.getArgument(0));

        action.execute(messageId);

        assertFalse(action.wasCalled());

        verify(outboxMessagePersistenceService).findUnpublishedMessageByIdWithErrors(messageId);
        verify(outboxMessagePersistenceService).existsNewerMessage(
                message.getEventType(),
                message.getAggregateId(),
                message.getCreatedAt()
        );
        verify(outboxMessagePersistenceService).save(argThat(saved ->
                saved.getStatus() == OutboxMessageStatus.CANCELED
        ));
        verifyNoMoreInteractions(outboxMessagePersistenceService);
    }

    @Test
    void execute_shouldRetryMessage_whenActionFailedAndMessageIsRetryable() {
        UUID messageId = UUID.randomUUID();

        OutboxMessage message = newMessage(
                messageId,
                OutboxMessageEventType.CREATE_EVENT,
                OutboxMessageStatus.IN_PROGRESS
        );

        RuntimeException error = new RuntimeException("temporary error");

        TestOutboxMessageAction action =
                new TestOutboxMessageAction(outboxMessagePersistenceService, error);

        setRetryProperties(action);

        when(outboxMessagePersistenceService.findUnpublishedMessageByIdWithErrors(messageId))
                .thenReturn(message);
        when(outboxMessagePersistenceService.save(any(OutboxMessage.class)))
                .thenAnswer(invocation -> invocation.getArgument(0));

        action.execute(messageId);

        assertTrue(action.wasCalled());

        verify(outboxMessagePersistenceService).findUnpublishedMessageByIdWithErrors(messageId);
        verify(outboxMessagePersistenceService).save(argThat(saved ->
                saved.getStatus() == OutboxMessageStatus.PENDING
                        && saved.getErrors() != null
                        && !saved.getErrors().isEmpty()
                        && "temporary error".equals(saved.getErrors().get(0).getErrorMessage())
                        && saved.getNextAttemptAt() != null
        ));
        verifyNoMoreInteractions(outboxMessagePersistenceService);
    }

    @Test
    void execute_shouldMarkFailed_whenMaxRetriesExceeded() {
        UUID messageId = UUID.randomUUID();

        OutboxMessage message = newMessage(
                messageId,
                OutboxMessageEventType.CREATE_EVENT,
                OutboxMessageStatus.IN_PROGRESS
        );

        for (int i = 0; i < 9; i++) {
            message.getErrors().add(new OutboxMessageError("old-error-" + i, message));
        }

        RuntimeException error = new RuntimeException("final error");

        TestOutboxMessageAction action =
                new TestOutboxMessageAction(outboxMessagePersistenceService, error);

        setRetryProperties(action);

        when(outboxMessagePersistenceService.findUnpublishedMessageByIdWithErrors(messageId))
                .thenReturn(message);
        when(outboxMessagePersistenceService.save(any(OutboxMessage.class)))
                .thenAnswer(invocation -> invocation.getArgument(0));

        action.execute(messageId);

        assertTrue(action.wasCalled());

        verify(outboxMessagePersistenceService).findUnpublishedMessageByIdWithErrors(messageId);
        verify(outboxMessagePersistenceService).save(argThat(saved ->
                saved.getStatus() == OutboxMessageStatus.FAILED
                        && saved.getErrors() != null
                        && !saved.getErrors().isEmpty()
                        && "final error".equals(saved.getErrors().get(saved.getErrors().size() - 1).getErrorMessage())
        ));
        verifyNoMoreInteractions(outboxMessagePersistenceService);
    }

    @Test
    void execute_shouldMarkFailed_whenEventIsNotRetryable() {
        UUID messageId = UUID.randomUUID();

        setEventTypeFlags(OutboxMessageEventType.CREATE_EVENT, false, false);

        OutboxMessage message = newMessage(
                messageId,
                OutboxMessageEventType.CREATE_EVENT,
                OutboxMessageStatus.IN_PROGRESS
        );

        RuntimeException error = new RuntimeException("non-retryable error");

        TestOutboxMessageAction action =
                new TestOutboxMessageAction(outboxMessagePersistenceService, error);

        setRetryProperties(action);

        when(outboxMessagePersistenceService.findUnpublishedMessageByIdWithErrors(messageId))
                .thenReturn(message);
        when(outboxMessagePersistenceService.save(any(OutboxMessage.class)))
                .thenAnswer(invocation -> invocation.getArgument(0));

        action.execute(messageId);

        assertTrue(action.wasCalled());

        verify(outboxMessagePersistenceService).findUnpublishedMessageByIdWithErrors(messageId);
        verify(outboxMessagePersistenceService).save(argThat(saved ->
                saved.getStatus() == OutboxMessageStatus.FAILED
                        && saved.getErrors() != null
                        && !saved.getErrors().isEmpty()
                        && "non-retryable error".equals(saved.getErrors().get(saved.getErrors().size() - 1).getErrorMessage())
        ));
        verifyNoMoreInteractions(outboxMessagePersistenceService);
    }

    private void setRetryProperties(OutboxMessageAction action) {
        ReflectionTestUtils.setField(action, "initialRetryDelay", 10);
        ReflectionTestUtils.setField(action, "maximumRetryDelay", 14400);
        ReflectionTestUtils.setField(action, "maximumOfRetries", 10);
    }

    private void setEventTypeFlags(OutboxMessageEventType eventType, boolean negligible, boolean retryable) {
        ReflectionTestUtils.setField(eventType, "negligible", negligible);
        ReflectionTestUtils.setField(eventType, "retryable", retryable);
    }

    private OutboxMessage newMessage(UUID messageId,
                                     OutboxMessageEventType eventType,
                                     OutboxMessageStatus status) {
        OutboxMessage message = new OutboxMessage(
                eventType,
                UUID.randomUUID(),
                UUID.randomUUID().toString(),
                "{\"test\":true}"
        );

        ReflectionTestUtils.setField(message, "uuid", messageId);
        ReflectionTestUtils.setField(message, "status", status);

        return message;
    }

    private static class TestOutboxMessageAction extends OutboxMessageAction {

        private final RuntimeException errorToThrow;
        private boolean called;

        private TestOutboxMessageAction(OutboxMessagePersistenceService service,
                                        RuntimeException errorToThrow) {
            super(service);
            this.errorToThrow = errorToThrow;
        }

        @Override
        public OutboxMessageEventType getEventType() {
            return OutboxMessageEventType.CREATE_EVENT;
        }

        @Override
        protected void action(OutboxMessage message) {
            called = true;
            if (errorToThrow != null) {
                throw errorToThrow;
            }
        }

        boolean wasCalled() {
            return called;
        }
    }
}
```
