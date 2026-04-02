```java
@ExtendWith(MockitoExtension.class)
class OutboxMessageProcessorTest {

    @Mock
    private OutboxMessagePersistenceService outboxMessagePersistenceService;

    @Mock
    private OutboxMessageDispatcher dispatcher;

    private OutboxMessageProcessor processor;

    @BeforeEach
    void setUp() {
        processor = new OutboxMessageProcessor(outboxMessagePersistenceService, dispatcher);
        ReflectionTestUtils.setField(processor, "visibilityTimeout", 300);
    }

    @Test
    void processOutboxMessage_shouldContinueProcessingBatchWhenFirstMessageHasTerminalRoutingError() {
        UUID createdBy = UUID.randomUUID();

        OutboxMessage first = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                createdBy,
                UUID.randomUUID().toString(),
                "{\"test\":1}"
        );
        ReflectionTestUtils.setField(first, "uuid", UUID.randomUUID());
        ReflectionTestUtils.setField(first, "status", OutboxMessageStatus.IN_PROGRESS);

        OutboxMessage second = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                createdBy,
                UUID.randomUUID().toString(),
                "{\"test\":2}"
        );
        ReflectionTestUtils.setField(second, "uuid", UUID.randomUUID());
        ReflectionTestUtils.setField(second, "status", OutboxMessageStatus.IN_PROGRESS);

        when(outboxMessagePersistenceService.findAndLockPendingMessagesToPublish(100, 300))
                .thenReturn(List.of(first, second));

        doThrow(new IllegalArgumentException("no action registered"))
                .when(dispatcher).dispatch(first);

        processor.processOutboxMessage();

        verify(dispatcher).dispatch(first);
        verify(dispatcher).dispatch(second);

        verify(outboxMessagePersistenceService).save(argThat(saved ->
                saved.getUuid().equals(first.getUuid())
                        && saved.getStatus() == OutboxMessageStatus.FAILED
                        && !saved.getErrors().isEmpty()
                        && "no action registered".equals(
                        saved.getErrors().get(saved.getErrors().size() - 1).getErrorMessage()
                )
        ));
    }

    @Test
    void processOutboxMessage_shouldNotFailMessageImmediatelyWhenUnexpectedExceptionEscapes() {
        UUID createdBy = UUID.randomUUID();

        OutboxMessage message = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                createdBy,
                UUID.randomUUID().toString(),
                "{\"test\":1}"
        );
        ReflectionTestUtils.setField(message, "uuid", UUID.randomUUID());
        ReflectionTestUtils.setField(message, "status", OutboxMessageStatus.IN_PROGRESS);

        when(outboxMessagePersistenceService.findAndLockPendingMessagesToPublish(100, 300))
                .thenReturn(List.of(message));

        doThrow(new IllegalStateException("unexpected failure"))
                .when(dispatcher).dispatch(message);

        processor.processOutboxMessage();

        verify(dispatcher).dispatch(message);
        verify(outboxMessagePersistenceService, never()).save(any());
    }

    @Test
    void processOutboxMessageById_shouldMarkMessageFailedWhenRoutingErrorOccurs() {
        UUID createdBy = UUID.randomUUID();
        UUID messageId = UUID.randomUUID();

        OutboxMessage message = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                createdBy,
                UUID.randomUUID().toString(),
                "{\"test\":1}"
        );
        ReflectionTestUtils.setField(message, "uuid", messageId);
        ReflectionTestUtils.setField(message, "status", OutboxMessageStatus.IN_PROGRESS);

        when(outboxMessagePersistenceService.findAndLockPendingMessageById(messageId))
                .thenReturn(message);

        doThrow(new IllegalArgumentException("no action registered"))
                .when(dispatcher).dispatch(message);

        processor.processOutboxMessage(messageId);

        verify(dispatcher).dispatch(message);
        verify(outboxMessagePersistenceService).save(argThat(saved ->
                saved.getUuid().equals(messageId)
                        && saved.getStatus() == OutboxMessageStatus.FAILED
                        && !saved.getErrors().isEmpty()
                        && "no action registered".equals(
                        saved.getErrors().get(saved.getErrors().size() - 1).getErrorMessage()
                )
        ));
    }

    @Test
    void processOutboxMessageById_shouldNotFailMessageImmediatelyWhenUnexpectedExceptionEscapes() {
        UUID createdBy = UUID.randomUUID();
        UUID messageId = UUID.randomUUID();

        OutboxMessage message = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                createdBy,
                UUID.randomUUID().toString(),
                "{\"test\":1}"
        );
        ReflectionTestUtils.setField(message, "uuid", messageId);
        ReflectionTestUtils.setField(message, "status", OutboxMessageStatus.IN_PROGRESS);

        when(outboxMessagePersistenceService.findAndLockPendingMessageById(messageId))
                .thenReturn(message);

        doThrow(new IllegalStateException("unexpected failure"))
                .when(dispatcher).dispatch(message);

        processor.processOutboxMessage(messageId);

        verify(dispatcher).dispatch(message);
        verify(outboxMessagePersistenceService, never()).save(any());
    }

    @Test
    void processOutboxMessageById_shouldDoNothingWhenMessageNotFound() {
        UUID messageId = UUID.randomUUID();

        when(outboxMessagePersistenceService.findAndLockPendingMessageById(messageId))
                .thenReturn(null);

        processor.processOutboxMessage(messageId);

        verify(dispatcher, never()).dispatch(any());
        verify(outboxMessagePersistenceService, never()).save(any());
    }
}
```
