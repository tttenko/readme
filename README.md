```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Map;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class OutboxMessageServiceTest {

    @Mock
    private OutboxMessagePersistenceService outboxMessagePersistenceService;

    private OutboxMessageService outboxMessageService;

    @BeforeEach
    void setUp() {
        ObjectMapper objectMapper = new ObjectMapper().findAndRegisterModules();
        outboxMessageService = new OutboxMessageService(outboxMessagePersistenceService, objectMapper);
    }

    @Test
    void addOutboxMessage_shouldSerializePayloadAndSaveMessage() {
        UUID createdBy = UUID.randomUUID();
        String aggregateId = UUID.randomUUID().toString();
        Map<String, Object> payload = Map.of(
                "vehicle_num", "A123AA77",
                "vehicle_name", "Камаз"
        );

        when(outboxMessagePersistenceService.save(any(OutboxMessage.class)))
                .thenAnswer(invocation -> invocation.getArgument(0));

        OutboxMessage result = outboxMessageService.addOutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                createdBy,
                aggregateId,
                payload
        );

        ArgumentCaptor<OutboxMessage> captor = ArgumentCaptor.forClass(OutboxMessage.class);
        verify(outboxMessagePersistenceService).save(captor.capture());

        OutboxMessage savedMessage = captor.getValue();

        assertNotNull(result);
        assertEquals(OutboxMessageEventType.CREATE_EVENT, savedMessage.getEventType());
        assertEquals(createdBy, savedMessage.getCreatedBy());
        assertEquals(aggregateId, savedMessage.getAggregateId());
        assertNotNull(savedMessage.getCreatedAt());
        assertNotNull(savedMessage.getNextAttemptAt());
        assertEquals(OutboxMessageStatus.PENDING, savedMessage.getStatus());

        assertNotNull(savedMessage.getPayload());
        assertTrue(savedMessage.getPayload().contains("\"vehicle_num\":\"A123AA77\""));
        assertTrue(savedMessage.getPayload().contains("\"vehicle_name\":\"Камаз\""));
    }
}

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.util.ReflectionTestUtils;
import ru.sber.cs.supplier.portal.core.eventshistory.client.EventsHistoryClient;
import ru.sber.cs.supplier.portal.core.eventshistory.dto.EventCreateDto;
import ru.sber.cs.supplier.portal.core.eventshistory.dto.EventParameterCreateDto;

import java.time.ZonedDateTime;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CreateEventOutboxMessageActionTest {

    @Mock
    private EventsHistoryClient eventsHistoryClient;

    @Mock
    private OutboxMessagePersistenceService outboxMessagePersistenceService;

    private CreateEventOutboxMessageAction action;
    private ObjectMapper objectMapper;

    @BeforeEach
    void setUp() {
        objectMapper = new ObjectMapper().findAndRegisterModules();
        action = new CreateEventOutboxMessageAction(
                eventsHistoryClient,
                outboxMessagePersistenceService,
                objectMapper
        );

        ReflectionTestUtils.setField(action, "initialRetryDelay", 10);
        ReflectionTestUtils.setField(action, "maximumRetryDelay", 14400);
        ReflectionTestUtils.setField(action, "maximumOfRetries", 10);
    }

    @Test
    void execute_shouldReadPayloadCallEventsHistoryClientAndMarkMessageDone() throws Exception {
        UUID messageId = UUID.randomUUID();
        UUID entityUuid = UUID.randomUUID();
        UUID createdBy = UUID.randomUUID();

        EventCreateDto payload = EventCreateDto.builder()
                .serviceId(StsHistoryEventIds.SERVICE_ID)
                .id(StsHistoryEventIds.STS_CREATE)
                .entityUuid(entityUuid)
                .submittedAt(ZonedDateTime.now())
                .submittedBy(createdBy.toString())
                .session("NO-SESSION")
                .userName("Иван Иванов")
                .userNode("NO-USERNODE")
                .parameters(List.of(
                        EventParameterCreateDto.builder()
                                .id("vehicle_num")
                                .value("A123AA77")
                                .build(),
                        EventParameterCreateDto.builder()
                                .id("vehicle_name")
                                .value("Камаз")
                                .build()
                ))
                .build();

        OutboxMessage message = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                createdBy,
                entityUuid.toString(),
                objectMapper.writeValueAsString(payload)
        );

        ReflectionTestUtils.setField(message, "uuid", messageId);
        ReflectionTestUtils.setField(message, "status", OutboxMessageStatus.IN_PROGRESS);

        when(outboxMessagePersistenceService.findUnpublishedMessageByIdWithErrors(messageId))
                .thenReturn(message);
        when(outboxMessagePersistenceService.save(any(OutboxMessage.class)))
                .thenAnswer(invocation -> invocation.getArgument(0));

        action.execute(messageId);

        ArgumentCaptor<EventCreateDto> eventCaptor = ArgumentCaptor.forClass(EventCreateDto.class);
        verify(eventsHistoryClient).createEvent(eventCaptor.capture());

        EventCreateDto sentEvent = eventCaptor.getValue();
        assertEquals(StsHistoryEventIds.SERVICE_ID, sentEvent.getServiceId());
        assertEquals(StsHistoryEventIds.STS_CREATE, sentEvent.getId());
        assertEquals(entityUuid, sentEvent.getEntityUuid());
        assertEquals(createdBy.toString(), sentEvent.getSubmittedBy());

        verify(outboxMessagePersistenceService).save(argThat(saved ->
                saved.getStatus() == OutboxMessageStatus.DONE
                        && saved.getPublishedAt() != null
        ));
    }
}

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.test.util.ReflectionTestUtils;
import ru.sber.cs.supplier.portal.core.eventshistory.client.EventsHistoryClient;
import ru.sber.cs.supplier.portal.core.eventshistory.dto.EventCreateDto;
import ru.sber.cs.supplier.portal.core.eventshistory.dto.EventParameterCreateDto;

import java.time.ZonedDateTime;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CreateEventOutboxMessageActionTest {

    @Mock
    private EventsHistoryClient eventsHistoryClient;

    @Mock
    private OutboxMessagePersistenceService outboxMessagePersistenceService;

    private CreateEventOutboxMessageAction action;
    private ObjectMapper objectMapper;

    @BeforeEach
    void setUp() {
        objectMapper = new ObjectMapper().findAndRegisterModules();
        action = new CreateEventOutboxMessageAction(
                eventsHistoryClient,
                outboxMessagePersistenceService,
                objectMapper
        );

        ReflectionTestUtils.setField(action, "initialRetryDelay", 10);
        ReflectionTestUtils.setField(action, "maximumRetryDelay", 14400);
        ReflectionTestUtils.setField(action, "maximumOfRetries", 10);
    }

    @Test
    void execute_shouldReadPayloadCallEventsHistoryClientAndMarkMessageDone() throws Exception {
        UUID messageId = UUID.randomUUID();
        UUID entityUuid = UUID.randomUUID();
        UUID createdBy = UUID.randomUUID();

        EventCreateDto payload = EventCreateDto.builder()
                .serviceId(StsHistoryEventIds.SERVICE_ID)
                .id(StsHistoryEventIds.STS_CREATE)
                .entityUuid(entityUuid)
                .submittedAt(ZonedDateTime.now())
                .submittedBy(createdBy.toString())
                .session("NO-SESSION")
                .userName("Иван Иванов")
                .userNode("NO-USERNODE")
                .parameters(List.of(
                        EventParameterCreateDto.builder()
                                .id("vehicle_num")
                                .value("A123AA77")
                                .build(),
                        EventParameterCreateDto.builder()
                                .id("vehicle_name")
                                .value("Камаз")
                                .build()
                ))
                .build();

        OutboxMessage message = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                createdBy,
                entityUuid.toString(),
                objectMapper.writeValueAsString(payload)
        );

        ReflectionTestUtils.setField(message, "uuid", messageId);
        ReflectionTestUtils.setField(message, "status", OutboxMessageStatus.IN_PROGRESS);

        when(outboxMessagePersistenceService.findUnpublishedMessageByIdWithErrors(messageId))
                .thenReturn(message);
        when(outboxMessagePersistenceService.save(any(OutboxMessage.class)))
                .thenAnswer(invocation -> invocation.getArgument(0));

        action.execute(messageId);

        ArgumentCaptor<EventCreateDto> eventCaptor = ArgumentCaptor.forClass(EventCreateDto.class);
        verify(eventsHistoryClient).createEvent(eventCaptor.capture());

        EventCreateDto sentEvent = eventCaptor.getValue();
        assertEquals(StsHistoryEventIds.SERVICE_ID, sentEvent.getServiceId());
        assertEquals(StsHistoryEventIds.STS_CREATE, sentEvent.getId());
        assertEquals(entityUuid, sentEvent.getEntityUuid());
        assertEquals(createdBy.toString(), sentEvent.getSubmittedBy());

        verify(outboxMessagePersistenceService).save(argThat(saved ->
                saved.getStatus() == OutboxMessageStatus.DONE
                        && saved.getPublishedAt() != null
        ));
    }
}
```
