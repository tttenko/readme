```java
@DataJpaTest
@Import(OutboxMessagePersistenceService.class)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@ActiveProfiles("test")
class OutboxMessageTest {

    @Autowired
    private OutboxMessagePersistenceService outboxMessagePersistenceService;

    @Autowired
    private OutboxMessageRepository outboxMessageRepository;

    @BeforeEach
    void setUp() {
        TimeZone.setDefault(TimeZone.getTimeZone("UTC"));
        outboxMessageRepository.deleteAll();
    }

    @Test
    void testStatusChange() throws Exception {
        OutboxMessage outboxMessage = createOutboxMessage(null);
        Field statusField = OutboxMessage.class.getDeclaredField("status");
        statusField.setAccessible(true);

        statusField.set(outboxMessage, OutboxMessageStatus.PENDING);
        outboxMessage.setDone();
        assertEquals(OutboxMessageStatus.PENDING, outboxMessage.getStatus());

        statusField.set(outboxMessage, OutboxMessageStatus.IN_PROGRESS);
        outboxMessage.setDone();
        assertEquals(OutboxMessageStatus.DONE, outboxMessage.getStatus());
        assertNotNull(outboxMessage.getPublishedAt());

        statusField.set(outboxMessage, OutboxMessageStatus.DONE);
        outboxMessage.setCanceled();
        assertEquals(OutboxMessageStatus.DONE, outboxMessage.getStatus());

        statusField.set(outboxMessage, OutboxMessageStatus.IN_PROGRESS);
        outboxMessage.setCanceled();
        assertEquals(OutboxMessageStatus.CANCELED, outboxMessage.getStatus());

        statusField.set(outboxMessage, OutboxMessageStatus.CANCELED);
        int errorsBeforeRetryFromCanceled = outboxMessage.getErrors().size();
        outboxMessage.retryExecution("Some error", 10);
        assertEquals(OutboxMessageStatus.CANCELED, outboxMessage.getStatus());
        assertEquals(errorsBeforeRetryFromCanceled, outboxMessage.getErrors().size());

        statusField.set(outboxMessage, OutboxMessageStatus.IN_PROGRESS);
        int errorsBeforeRetryFromInProgress = outboxMessage.getErrors().size();
        outboxMessage.retryExecution("Some error", 10);
        assertEquals(OutboxMessageStatus.PENDING, outboxMessage.getStatus());
        assertEquals(errorsBeforeRetryFromInProgress + 1, outboxMessage.getErrors().size());
        assertNotNull(outboxMessage.getNextAttemptAt());

        statusField.set(outboxMessage, OutboxMessageStatus.DONE);
        int errorsBeforeFailFromDone = outboxMessage.getErrors().size();
        outboxMessage.setFailed("Some error");
        assertEquals(OutboxMessageStatus.DONE, outboxMessage.getStatus());
        assertEquals(errorsBeforeFailFromDone, outboxMessage.getErrors().size());

        statusField.set(outboxMessage, OutboxMessageStatus.IN_PROGRESS);
        int errorsBeforeFailFromInProgress = outboxMessage.getErrors().size();
        outboxMessage.setFailed("Some error");
        assertEquals(OutboxMessageStatus.FAILED, outboxMessage.getStatus());
        assertEquals(errorsBeforeFailFromInProgress + 1, outboxMessage.getErrors().size());
    }

    @Test
    void symmetry() throws Exception {
        UUID uuid = UUID.randomUUID();
        OutboxMessage outboxMessage1 = createOutboxMessage(uuid);
        OutboxMessage outboxMessage2 = createOutboxMessage(uuid);

        assertEquals(outboxMessage1, outboxMessage2);
        assertEquals(outboxMessage2, outboxMessage1);
    }

    @Test
    void transitivity() throws Exception {
        UUID uuid = UUID.randomUUID();
        OutboxMessage outboxMessage1 = createOutboxMessage(uuid);
        OutboxMessage outboxMessage2 = createOutboxMessage(uuid);
        OutboxMessage outboxMessage3 = createOutboxMessage(uuid);

        assertEquals(outboxMessage1, outboxMessage2);
        assertEquals(outboxMessage2, outboxMessage3);
        assertEquals(outboxMessage1, outboxMessage3);
    }

    @Test
    void nullHandling() throws Exception {
        OutboxMessage outboxMessage = createOutboxMessage(UUID.randomUUID());
        assertNotEquals(null, outboxMessage);
    }

    @Test
    void differentClasses() throws Exception {
        OutboxMessage outboxMessage = createOutboxMessage(UUID.randomUUID());
        Object otherObject = new Object();

        assertNotEquals(outboxMessage, otherObject);
    }

    @Test
    void testEqualsDifferentUuids() throws Exception {
        OutboxMessage outboxMessage1 = createOutboxMessage(UUID.randomUUID());
        OutboxMessage outboxMessage2 = createOutboxMessage(UUID.randomUUID());

        assertNotEquals(outboxMessage1, outboxMessage2);
    }

    @Test
    void hibernateProxy() throws Exception {
        OutboxMessage outboxMessage =
                outboxMessagePersistenceService.save(createOutboxMessage(null));

        OutboxMessage proxy =
                outboxMessageRepository.getReferenceById(outboxMessage.getUuid());

        assertEquals(outboxMessage, proxy);
        assertEquals(proxy, outboxMessage);
    }

    @Test
    void consistency() throws Exception {
        OutboxMessage outboxMessage = createOutboxMessage(UUID.randomUUID());

        int hashCode1 = outboxMessage.hashCode();
        int hashCode2 = outboxMessage.hashCode();

        assertEquals(hashCode1, hashCode2);
    }

    @Test
    void equalsObjectsHaveSameHashCode() throws Exception {
        UUID uuid = UUID.randomUUID();
        OutboxMessage outboxMessage1 = createOutboxMessage(uuid);
        OutboxMessage outboxMessage2 = createOutboxMessage(uuid);

        assertEquals(outboxMessage1.hashCode(), outboxMessage2.hashCode());
    }

    private OutboxMessage createOutboxMessage(UUID uuid) throws Exception {
        OutboxMessage outboxMessage = new OutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                UUID.randomUUID(),
                "SOME_AGGREGATE_ID",
                "{\"some\":\"payload\"}"
        );

        if (uuid != null) {
            Field uuidField = BaseEntity.class.getDeclaredField("uuid");
            uuidField.setAccessible(true);
            uuidField.set(outboxMessage, uuid);
        }

        return outboxMessage;
    }
}
```
