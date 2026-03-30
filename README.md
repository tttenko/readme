```java
@Getter
@AllArgsConstructor
public enum OutboxMessageEventType {

    CREATE_EVENT(false, true);

    private final boolean negligible;
    private final boolean retryable;
}

public enum OutboxMessageStatus {
    PENDING,
    IN_PROGRESS,
    DONE,
    CANCELED,
    FAILED
}

@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor(access = AccessLevel.PROTECTED)
@MappedSuperclass
public abstract class BaseEntity implements Serializable {

    @Serial
    private static final long serialVersionUID = 2765238233714717629L;

    @Id
    @Column(name = "uuid", unique = true, nullable = false)
    @GeneratedValue(strategy = GenerationType.UUID)
    protected UUID uuid;

    @Override
    @SuppressWarnings("squid:S2097")
    public final boolean equals(Object object) {
        if (this == object) {
            return true;
        }
        if (object == null) {
            return false;
        }

        Class<?> objectEffectiveClass = object instanceof HibernateProxy proxy
                ? proxy.getHibernateLazyInitializer().getPersistentClass()
                : object.getClass();

        Class<?> thisEffectiveClass = this instanceof HibernateProxy proxy
                ? proxy.getHibernateLazyInitializer().getPersistentClass()
                : this.getClass();

        if (thisEffectiveClass != objectEffectiveClass) {
            return false;
        }

        BaseEntity basicEntity = (BaseEntity) object;
        return getUuid() != null && Objects.equals(getUuid(), basicEntity.getUuid());
    }

    @Override
    public final int hashCode() {
        return this instanceof HibernateProxy proxy
                ? proxy.getHibernateLazyInitializer().getPersistentClass().hashCode()
                : getClass().hashCode();
    }
}

@Slf4j
@Entity
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Table(name = "OUTBOX_MESSAGE")
public class OutboxMessage extends BaseEntity {

    private static final String STATUS_CHANGE_ERROR =
            "Unable to change outbox message status to '{}', message is already processed, uuid: '{}', event type: '{}', aggregate id: '{}', status: '{}'";

    @Column(name = "STATUS", nullable = false)
    @Enumerated(EnumType.STRING)
    private OutboxMessageStatus status = PENDING;

    @Column(name = "EVENT_TYPE", nullable = false)
    @Enumerated(EnumType.STRING)
    private OutboxMessageEventType eventType;

    @Column(name = "AGGREGATE_ID", nullable = false)
    private String aggregateId;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "PAYLOAD", columnDefinition = "jsonb")
    private String payload;

    @Column(name = "CREATED_AT", nullable = false)
    private ZonedDateTime createdAt;

    @Column(name = "CREATED_BY", nullable = false)
    private UUID createdBy;

    @Column(name = "PUBLISHED_AT")
    private ZonedDateTime publishedAt;

    @Column(name = "PICKED_AT")
    private ZonedDateTime pickedAt;

    @Column(name = "NEXT_ATTEMPT_AT")
    private ZonedDateTime nextAttemptAt = ZonedDateTime.now();

    @OneToMany(mappedBy = "outboxMessage", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OutboxMessageError> errors = new ArrayList<>();

    public OutboxMessage(
            @NotNull OutboxMessageEventType eventType,
            @NotNull UUID createdBy,
            @NotBlank String aggregateId,
            @NotBlank String payload
    ) {
        this.eventType = eventType;
        this.aggregateId = aggregateId;
        this.payload = payload;
        this.createdAt = ZonedDateTime.now();
        this.createdBy = createdBy;
    }

    public void setDone() {
        if (status != IN_PROGRESS) {
            log.warn(STATUS_CHANGE_ERROR, DONE, getUuid(), eventType, aggregateId, status);
            return;
        }

        publishedAt = ZonedDateTime.now();
        status = DONE;
    }

    public void setCanceled() {
        if (status != PENDING && status != IN_PROGRESS) {
            log.warn(STATUS_CHANGE_ERROR, CANCELED, getUuid(), eventType, aggregateId, status);
            return;
        }

        status = CANCELED;
    }

    public void retryExecution(@NotBlank String error, long delay) {
        if (status != IN_PROGRESS) {
            log.warn(STATUS_CHANGE_ERROR, PENDING, getUuid(), eventType, aggregateId, status);
            return;
        }

        status = PENDING;
        errors.add(new OutboxMessageError(error, this));
        nextAttemptAt = ZonedDateTime.now().plus(Duration.ofSeconds(delay));
    }

    public void setFailed(@NotBlank String error) {
        if (status != IN_PROGRESS) {
            log.warn(STATUS_CHANGE_ERROR, FAILED, getUuid(), eventType, aggregateId, status);
            return;
        }

        errors.add(new OutboxMessageError(error, this));
        status = FAILED;
    }
}

@Entity
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Table(name = "OUTBOX_MESSAGE_ERROR")
public class OutboxMessageError extends BaseEntity {

    @Column(name = "ATTEMPTED_AT")
    private ZonedDateTime attemptedAt = ZonedDateTime.now();

    @Column(name = "ERROR_MESSAGE", columnDefinition = "TEXT")
    private String errorMessage;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "OUTBOX_MESSAGE_UUID", nullable = false)
    private OutboxMessage outboxMessage;

    public OutboxMessageError(
            @NotBlank String errorMessage,
            @NotNull OutboxMessage outboxMessage
    ) {
        this.errorMessage = errorMessage;
        this.outboxMessage = outboxMessage;
    }
}

public interface OutboxMessageRepository extends JpaRepository<OutboxMessage, UUID>,
        OutboxMessageRepositoryCustom {

    @Query("""
        SELECT COUNT(m) > 0 FROM OutboxMessage m
        WHERE m.eventType = :eventType
        AND m.aggregateId = :aggregateId
        AND m.createdAt > :createdAt
    """)
    boolean existsNewerMessage(
            @Param("eventType") OutboxMessageEventType eventType,
            @Param("aggregateId") String aggregateId,
            @Param("createdAt") ZonedDateTime createdAt
    );

    @Query("""
        SELECT DISTINCT om FROM OutboxMessage om
        LEFT JOIN FETCH om.errors
        WHERE om.publishedAt IS NULL
        ORDER BY om.createdAt
    """)
    List<OutboxMessage> findUnpublishedMessagesWithErrors(Pageable pageable);

    @Query("""
        SELECT om FROM OutboxMessage om
        LEFT JOIN FETCH om.errors
        WHERE om.uuid = :uuid
    """)
    OutboxMessage findUnpublishedMessageByIdWithErrors(@Param("uuid") UUID uuid);
}

public interface OutboxMessageRepository extends JpaRepository<OutboxMessage, UUID>,
        OutboxMessageRepositoryCustom {

    @Query("""
        SELECT COUNT(m) > 0 FROM OutboxMessage m
        WHERE m.eventType = :eventType
        AND m.aggregateId = :aggregateId
        AND m.createdAt > :createdAt
    """)
    boolean existsNewerMessage(
            @Param("eventType") OutboxMessageEventType eventType,
            @Param("aggregateId") String aggregateId,
            @Param("createdAt") ZonedDateTime createdAt
    );

    @Query("""
        SELECT DISTINCT om FROM OutboxMessage om
        LEFT JOIN FETCH om.errors
        WHERE om.publishedAt IS NULL
        ORDER BY om.createdAt
    """)
    List<OutboxMessage> findUnpublishedMessagesWithErrors(Pageable pageable);

    @Query("""
        SELECT om FROM OutboxMessage om
        LEFT JOIN FETCH om.errors
        WHERE om.uuid = :uuid
    """)
    OutboxMessage findUnpublishedMessageByIdWithErrors(@Param("uuid") UUID uuid);
}

public interface OutboxMessageRepositoryCustom {

    List<OutboxMessage> findAndLockPendingMessagesToPublish(int limit, int visibilityTimeout);

    OutboxMessage findAndLockPendingMessageById(UUID uuid);

    int deleteOldProcessedMessages(int daysToLive, int batchSize);
}

@Service
@RequiredArgsConstructor
public class OutboxMessagePersistenceService {

    private final OutboxMessageRepository outboxMessageRepository;

    @Transactional
    public OutboxMessage findAndLockPendingMessageById(UUID uuid) {
        return outboxMessageRepository.findAndLockPendingMessageById(uuid);
    }

    @Transactional
    public List<OutboxMessage> findAndLockPendingMessagesToPublish(int limit, int visibilityTimeout) {
        return outboxMessageRepository.findAndLockPendingMessagesToPublish(limit, visibilityTimeout);
    }

    @Transactional
    public List<OutboxMessage> findUnpublishedMessagesWithErrors(int limit) {
        return outboxMessageRepository.findUnpublishedMessagesWithErrors(PageRequest.of(0, limit));
    }

    @Transactional
    public OutboxMessage findUnpublishedMessageByIdWithErrors(UUID id) {
        OutboxMessage message = outboxMessageRepository.findUnpublishedMessageByIdWithErrors(id);
        if (message == null) {
            throw new EntityNotFoundException("Outbox message not found: " + id);
        }
        return message;
    }

    @Transactional
    public OutboxMessage save(OutboxMessage message) {
        return outboxMessageRepository.save(message);
    }

    @Transactional
    public int deleteOldProcessedMessages(int daysToLive, int batchSize) {
        return outboxMessageRepository.deleteOldProcessedMessages(daysToLive, batchSize);
    }

    @Transactional
    public boolean existsNewerMessage(OutboxMessageEventType eventType,
                                      String aggregateId,
                                      ZonedDateTime createdAt) {
        return outboxMessageRepository.existsNewerMessage(eventType, aggregateId, createdAt);
    }
}

public class OutboxMessageActionException extends RuntimeException {

    public OutboxMessageActionException(String message, Throwable cause) {
        super(message, cause);
    }
}

@Slf4j
@RequiredArgsConstructor
public abstract class OutboxMessageAction {

    @Value("${outbox.processing.delay_sec:10}")
    private int initialRetryDelay;

    @Value("${outbox.processing.max-delay_sec:14400}")
    private int maximumRetryDelay;

    @Value("${outbox.processing.max-attempts:10}")
    private int maximumOfRetries;

    private final OutboxMessagePersistenceService outboxMessagePersistenceService;

    @SuppressWarnings({"java:S2245", "java:S2140"})
    public void execute(UUID messageId) {
        OutboxMessage message =
                outboxMessagePersistenceService.findUnpublishedMessageByIdWithErrors(messageId);

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

        try {
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

                log.error(String.format(
                        "Failed to process outbox message: UUID '%s', event '%s', aggregate id '%s'",
                        message.getUuid(), message.getEventType(), message.getAggregateId()
                ), error);

                message.setFailed(error.getMessage());

            } else {
                long baseDelay = initialRetryDelay * (long) Math.pow(2, message.getErrors().size());
                long jitterDelay = (long) (baseDelay * (0.8 + 0.4 * Math.random()));

                message.retryExecution(
                        error.getMessage(),
                        Math.min(jitterDelay, maximumRetryDelay)
                );

                log.error(String.format(
                        "Outbox message processing failed. The message will be automatically retried: UUID '%s', event '%s', aggregate id '%s', next attempt at '%s'",
                        message.getUuid(),
                        message.getEventType(),
                        message.getAggregateId(),
                        message.getNextAttemptAt()
                ), error);
            }

            outboxMessagePersistenceService.save(message);
        }
    }

    public abstract OutboxMessageEventType getEventType();

    protected abstract void action(OutboxMessage message) throws JsonProcessingException;
}

@Slf4j
@Component
public class CreateEventOutboxMessageAction extends OutboxMessageAction {

    private final EventsHistoryClient eventsHistoryClient;
    private final ObjectMapper objectMapper;

    public CreateEventOutboxMessageAction(
            @NotNull EventsHistoryClient eventsHistoryClient,
            @NotNull OutboxMessagePersistenceService outboxMessagePersistenceService,
            @NotNull ObjectMapper objectMapper
    ) {
        super(outboxMessagePersistenceService);
        this.eventsHistoryClient = eventsHistoryClient;
        this.objectMapper = objectMapper;
    }

    @Override
    public OutboxMessageEventType getEventType() {
        return OutboxMessageEventType.CREATE_EVENT;
    }

    @Override
    protected void action(OutboxMessage message) throws JsonProcessingException {
        EventCreateDto payload = objectMapper.readValue(message.getPayload(), EventCreateDto.class);
        eventsHistoryClient.createEvent(payload);
    }
}

@Service
@RequiredArgsConstructor
public class OutboxMessageActionRegistry {

    private final List<OutboxMessageAction> actions;

    private final Map<OutboxMessageEventType, OutboxMessageAction> actionMap =
            new EnumMap<>(OutboxMessageEventType.class);

    @PostConstruct
    public void init() {
        for (OutboxMessageAction action : actions) {
            actionMap.put(action.getEventType(), action);
        }
    }

    public OutboxMessageAction getAction(@NotNull OutboxMessageEventType eventType) {
        OutboxMessageAction action = actionMap.get(eventType);

        if (action == null) {
            throw new IllegalArgumentException(
                    String.format("No outbox message action registered for event '%s'", eventType)
            );
        }

        return action;
    }
}

@Service
@RequiredArgsConstructor
public class OutboxMessageDispatcher {

    private final OutboxMessageActionRegistry registry;

    public void dispatch(@NotNull OutboxMessage message) {
        OutboxMessageAction action = registry.getAction(message.getEventType());
        action.execute(message.getUuid());
    }
}

public record OutboxMessageEvent(@NotNull UUID uuid) implements Serializable {
}

@Component
@RequiredArgsConstructor
public class OutboxMessageEventListener {

    private final OutboxMessageProcessor outboxMessageProcessor;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void outboxMessageHandler(OutboxMessageEvent outboxMessageEvent) {
        outboxMessageProcessor.processOutboxMessage(outboxMessageEvent.uuid());
    }
}

@Slf4j
@Service
@RequiredArgsConstructor
public class OutboxMessageProcessor {

    @Value("${outbox.processing.visibility_timeout_sec:300}")
    private int visibilityTimeout;

    private final OutboxMessagePersistenceService outboxMessagePersistenceService;
    private final OutboxMessageDispatcher dispatcher;

    @Scheduled(fixedDelayString = "${outbox.processing.delay_sec:10}000")
    public void processOutboxMessage() {
        List<OutboxMessage> messages =
                outboxMessagePersistenceService.findAndLockPendingMessagesToPublish(100, visibilityTimeout);

        messages.forEach(dispatcher::dispatch);
    }

    public void processOutboxMessage(UUID outboxMessageId) {
        OutboxMessage outboxMessage =
                outboxMessagePersistenceService.findAndLockPendingMessageById(outboxMessageId);

        if (outboxMessage != null) {
            dispatcher.dispatch(outboxMessage);
        }
    }
}

@Service
@RequiredArgsConstructor
public class OutboxMessageCleaningService {

    @Value("${outbox.cleanup.days_to_live:90}")
    private int daysToLive;

    @Value("${outbox.cleanup.batch_size:1000}")
    private int batchSize;

    private final OutboxMessagePersistenceService outboxMessagePersistenceService;

    @Scheduled(cron = "${outbox.cleanup.cron:0 0 3 * * ?}")
    public void cleanOutboxMessage() throws InterruptedException {
        int deletedCount;

        do {
            deletedCount = outboxMessagePersistenceService
                    .deleteOldProcessedMessages(daysToLive, batchSize);

            if (deletedCount > 0) {
                Thread.sleep(1000);
            }
        } while (deletedCount > 0);
    }
}

@Service
@RequiredArgsConstructor
public class OutboxMessageService {

    private final OutboxMessagePersistenceService outboxMessagePersistenceService;
    private final ObjectMapper objectMapper;

    @Transactional
    public OutboxMessage addOutboxMessage(
            @NotNull OutboxMessageEventType eventType,
            @NotNull UUID createdBy,
            @NotBlank String aggregateId,
            @NotNull Object payload
    ) {
        OutboxMessage message;

        try {
            message = new OutboxMessage(
                    eventType,
                    createdBy,
                    aggregateId,
                    objectMapper.writeValueAsString(payload)
            );
        } catch (JsonProcessingException e) {
            throw new OutboxMessageActionException(
                    String.format(
                            "Unable to serialize payload for outbox message: event '%s', aggregate id '%s'",
                            eventType, aggregateId
                    ),
                    e
            );
        }

        return outboxMessagePersistenceService.save(message);
    }
}

@Service
@RequiredArgsConstructor
public class OutboxMessageService {

    private final OutboxMessagePersistenceService outboxMessagePersistenceService;
    private final ObjectMapper objectMapper;

    @Transactional
    public OutboxMessage addOutboxMessage(
            @NotNull OutboxMessageEventType eventType,
            @NotNull UUID createdBy,
            @NotBlank String aggregateId,
            @NotNull Object payload
    ) {
        OutboxMessage message;

        try {
            message = new OutboxMessage(
                    eventType,
                    createdBy,
                    aggregateId,
                    objectMapper.writeValueAsString(payload)
            );
        } catch (JsonProcessingException e) {
            throw new OutboxMessageActionException(
                    String.format(
                            "Unable to serialize payload for outbox message: event '%s', aggregate id '%s'",
                            eventType, aggregateId
                    ),
                    e
            );
        }

        return outboxMessagePersistenceService.save(message);
    }
}
```
