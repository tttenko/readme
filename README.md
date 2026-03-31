```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.util.ReflectionTestUtils;

import java.util.List;
import java.util.UUID;

import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class OutboxMessageProcessorTest {

    @Mock
    private OutboxMessagePersistenceService outboxMessagePersistenceService;

    @Mock
    private OutboxMessageDispatcher dispatcher;

    private OutboxMessageProcessor outboxMessageProcessor;

    @BeforeEach
    void setUp() {
        outboxMessageProcessor = new OutboxMessageProcessor(
                outboxMessagePersistenceService,
                dispatcher
        );

        ReflectionTestUtils.setField(outboxMessageProcessor, "visibilityTimeout", 300);
    }

    @Test
    void processOutboxMessage_shouldDispatchAllLockedMessages() {
        OutboxMessage message1 = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                UUID.randomUUID(),
                "aggregate-1",
                "{\"key\":\"value1\"}"
        );

        OutboxMessage message2 = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                UUID.randomUUID(),
                "aggregate-2",
                "{\"key\":\"value2\"}"
        );

        when(outboxMessagePersistenceService.findAndLockPendingMessagesToPublish(100, 300))
                .thenReturn(List.of(message1, message2));

        outboxMessageProcessor.processOutboxMessage();

        verify(dispatcher).dispatch(message1);
        verify(dispatcher).dispatch(message2);
        verifyNoMoreInteractions(dispatcher);
    }

    @Test
    void processOutboxMessageById_shouldDispatchWhenMessageFound() {
        UUID messageId = UUID.randomUUID();

        OutboxMessage message = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                UUID.randomUUID(),
                "aggregate-1",
                "{\"key\":\"value1\"}"
        );

        when(outboxMessagePersistenceService.findAndLockPendingMessageById(messageId))
                .thenReturn(message);

        outboxMessageProcessor.processOutboxMessage(messageId);

        verify(dispatcher).dispatch(message);
    }

    @Test
    void processOutboxMessageById_shouldNotDispatchWhenMessageNotFound() {
        UUID messageId = UUID.randomUUID();

        when(outboxMessagePersistenceService.findAndLockPendingMessageById(messageId))
                .thenReturn(null);

        outboxMessageProcessor.processOutboxMessage(messageId);

        verifyNoInteractions(dispatcher);
    }
}
```
