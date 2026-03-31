```java
@ExtendWith(MockitoExtension.class)
class OutboxMessageActionRegistryTest {

    @Mock
    private OutboxMessageAction createEventAction;

    @Mock
    private OutboxMessageAction sendTrackerHistoryAction;

    private OutboxMessageActionRegistry registry;

    @BeforeEach
    void setUp() {
        when(createEventAction.getEventType()).thenReturn(OutboxMessageEventType.CREATE_EVENT);
        when(sendTrackerHistoryAction.getEventType()).thenReturn(OutboxMessageEventType.SEND_TRACKER_HISTORY);

        registry = new OutboxMessageActionRegistry(
                List.of(createEventAction, sendTrackerHistoryAction)
        );

        registry.init();
    }

    @Test
    void getAction_shouldReturnCreateEventAction() {
        OutboxMessageAction result = registry.getAction(OutboxMessageEventType.CREATE_EVENT);

        assertSame(createEventAction, result);
    }

    @Test
    void getAction_shouldReturnSendTrackerHistoryAction() {
        OutboxMessageAction result = registry.getAction(OutboxMessageEventType.SEND_TRACKER_HISTORY);

        assertSame(sendTrackerHistoryAction, result);
    }

    @Test
    void getAction_shouldThrowException_whenActionNotRegistered() {
        OutboxMessageActionRegistry emptyRegistry =
                new OutboxMessageActionRegistry(Collections.emptyList());

        emptyRegistry.init();

        IllegalArgumentException exception = assertThrows(
                IllegalArgumentException.class,
                () -> emptyRegistry.getAction(OutboxMessageEventType.CREATE_EVENT)
        );

        assertEquals(
                "No outbox message action registered for event 'CREATE_EVENT'",
                exception.getMessage()
        );
    }
}

@ExtendWith(MockitoExtension.class)
class OutboxMessageCleaningServiceTest {

    @Mock
    private OutboxMessagePersistenceService outboxMessagePersistenceService;

    @InjectMocks
    private OutboxMessageCleaningService outboxMessageCleaningService;

    @BeforeEach
    void setUp() {
        ReflectionTestUtils.setField(outboxMessageCleaningService, "daysToLive", 90);
        ReflectionTestUtils.setField(outboxMessageCleaningService, "batchSize", 1000);
    }

    @Test
    void cleanOutboxMessage_shouldCallDeleteOnce_whenNothingToDelete() throws Exception {
        when(outboxMessagePersistenceService.deleteOldProcessedMessages(90, 1000))
                .thenReturn(0);

        outboxMessageCleaningService.cleanOutboxMessage();

        verify(outboxMessagePersistenceService).deleteOldProcessedMessages(90, 1000);
        verifyNoMoreInteractions(outboxMessagePersistenceService);
    }

    @Test
    void cleanOutboxMessage_shouldRepeatUntilNoMessagesLeft() throws Exception {
        when(outboxMessagePersistenceService.deleteOldProcessedMessages(90, 1000))
                .thenReturn(1, 0);

        outboxMessageCleaningService.cleanOutboxMessage();

        verify(outboxMessagePersistenceService, times(2))
                .deleteOldProcessedMessages(90, 1000);
        verifyNoMoreInteractions(outboxMessagePersistenceService);
    }

    @Test
    void cleanOutboxMessage_shouldThrowInterruptedException_whenSleepInterrupted() {
        when(outboxMessagePersistenceService.deleteOldProcessedMessages(90, 1000))
                .thenReturn(1);

        Thread.currentThread().interrupt();

        assertThrows(
                InterruptedException.class,
                () -> outboxMessageCleaningService.cleanOutboxMessage()
        );

        verify(outboxMessagePersistenceService).deleteOldProcessedMessages(90, 1000);
        verifyNoMoreInteractions(outboxMessagePersistenceService);
    }
}

@ExtendWith(MockitoExtension.class)
class OutboxMessageDispatcherTest {

    @Mock
    private OutboxMessageActionRegistry registry;

    @Mock
    private OutboxMessageAction action;

    @InjectMocks
    private OutboxMessageDispatcher dispatcher;

    @Test
    void dispatch_shouldGetActionFromRegistryAndExecuteIt() {
        UUID messageUuid = UUID.randomUUID();

        OutboxMessage message = mock(OutboxMessage.class);
        when(message.getUuid()).thenReturn(messageUuid);
        when(message.getEventType()).thenReturn(OutboxMessageEventType.CREATE_EVENT);

        when(registry.getAction(OutboxMessageEventType.CREATE_EVENT)).thenReturn(action);

        dispatcher.dispatch(message);

        verify(registry).getAction(OutboxMessageEventType.CREATE_EVENT);
        verify(action).execute(messageUuid);
        verifyNoMoreInteractions(registry, action);
    }
}

@ExtendWith(MockitoExtension.class)
class OutboxMessageEventListenerTest {

    @Mock
    private OutboxMessageProcessor outboxMessageProcessor;

    @InjectMocks
    private OutboxMessageEventListener outboxMessageEventListener;

    @Test
    void outboxMessageHandler_shouldCallProcessorWithEventUuid() {
        UUID messageUuid = UUID.randomUUID();
        OutboxMessageEvent event = new OutboxMessageEvent(messageUuid);

        outboxMessageEventListener.outboxMessageHandler(event);

        verify(outboxMessageProcessor).processOutboxMessage(messageUuid);
        verifyNoMoreInteractions(outboxMessageProcessor);
    }
}
```
