```java
@Service
@RequiredArgsConstructor
public class StsEventsHistoryService {

    private static final String NO_SESSION = "NO-SESSION";
    private static final String NO_USER_NODE = "NO-USERNODE";
    private static final String NO_USER = "NO-USER";

    private final OutboxMessageService outboxMessageService;
    private final ApplicationEventPublisher applicationEventPublisher;

    /** Сохраняет событие создания СТС в outbox для дальнейшей отправки в историю. */
    public void sendCreateEvent(StsDataEntity entity) {
        EventCreateDto payload = EventCreateDto.builder()
                .serviceId(StsHistoryEventIds.SERVICE_ID)
                .id(StsHistoryEventIds.STS_CREATE)
                .entityUuid(entity.getUuid())
                .submittedAt(entity.getCreatedAt())
                .submittedBy(entity.getCreatedBy().toString())
                .session(NO_SESSION)
                .userName(resolveFullName())
                .userNode(NO_USER_NODE)
                .parameters(List.of(
                        parameter("vehicle_num", entity.getVehicleNumber()),
                        parameter("vehicle_name", entity.getVehicleBrand())
                ))
                .build();

        OutboxMessage message = outboxMessageService.addOutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                entity.getCreatedBy(),
                entity.getUuid().toString(),
                payload
        );

        applicationEventPublisher.publishEvent(new OutboxMessageEvent(message.getUuid()));
    }

    private EventParameterCreateDto parameter(String id, String value) {
        return EventParameterCreateDto.builder()
                .id(id)
                .value(value)
                .build();
    }

    private String resolveFullName() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

        if (authentication == null || authentication.getPrincipal() == null) {
            return NO_USER;
        }

        Object principal = authentication.getPrincipal();

        if (principal instanceof AuthorizedUser authorizedUser
                && StringUtils.hasText(authorizedUser.getFullName())) {
            return authorizedUser.getFullName();
        }

        return NO_USER;
    }
}

@ExceptionHandler(OutboxMessageActionException.class)
@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
public ResponseEntity<Object> handleOutboxMessageActionException(
        OutboxMessageActionException ex,
        WebRequest request
) {
    log.error("Outbox processing preparation failed: {}", ex.getMessage(), ex);

    return createResponseEntity(
            "Внутренняя ошибка при подготовке события для отправки",
            new HttpHeaders(),
            HttpStatus.INTERNAL_SERVER_ERROR,
            request
    );
}

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Value("${async-executor.core-pool-size:5}")
    private int asyncExecutorCorePoolSize;

    @Value("${async-executor.max-pool-size:20}")
    private int asyncExecutorMaxPoolSize;

    @Value("${async-executor.queue-capacity:200}")
    private int asyncExecutorQueueCapacity;

    @Value("${async-executor.await-termination-sec:30}")
    private int asyncExecutorAwaitTermination;

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(asyncExecutorCorePoolSize);
        executor.setMaxPoolSize(asyncExecutorMaxPoolSize);
        executor.setQueueCapacity(asyncExecutorQueueCapacity);
        executor.setThreadNamePrefix("AsyncEvent-");
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(asyncExecutorAwaitTermination);
        executor.initialize();
        return executor;
    }
}

@Configuration
@EnableScheduling
public class SchedulingConfig {
}

async-executor:
  core-pool-size: ${app.executor.core-pool-size:5}
  max-pool-size: ${app.executor.max-pool-size:20}
  queue-capacity: ${app.executor.queue-capacity:200}
  await-termination-sec: 30

outbox:
  processing:
    delay_sec: 10
    max-delay_sec: 14400
    max-attempts: 10
    visibility_timeout_sec: 300
  cleanup:
    cron: "0 0 3 * * ?"
    days_to_live: 90
    batch_size: 1000


<?xml version="1.1" encoding="UTF-8" standalone="no"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.6.xsd">

    <changeSet id="CONTROL-VEHICLE/create_outbox_message/10" author="OpenAI">
        <preConditions onFail="HALT" onError="HALT">
            <not>
                <tableExists tableName="outbox_message"/>
            </not>
        </preConditions>
        <sql>
            CREATE TABLE OUTBOX_MESSAGE (
                UUID            UUID NOT NULL,
                EVENT_TYPE      VARCHAR(255) NOT NULL,
                AGGREGATE_ID    VARCHAR(255) NOT NULL,
                PAYLOAD         JSONB,
                CREATED_AT      TIMESTAMP WITH TIME ZONE NOT NULL,
                CREATED_BY      UUID NOT NULL,
                PUBLISHED_AT    TIMESTAMP WITH TIME ZONE,
                NEXT_ATTEMPT_AT TIMESTAMP WITH TIME ZONE NOT NULL,
                PICKED_AT       TIMESTAMP WITH TIME ZONE,
                STATUS          VARCHAR(50) NOT NULL,

                CONSTRAINT PK_OUTBOX_MESSAGE PRIMARY KEY (UUID)
            );
        </sql>
    </changeSet>

    <changeSet id="CONTROL-VEHICLE/create_outbox_message/20" author="OpenAI">
        <preConditions onFail="HALT" onError="HALT">
            <not>
                <tableExists tableName="outbox_message_error"/>
            </not>
        </preConditions>
        <sql>
            CREATE TABLE OUTBOX_MESSAGE_ERROR (
                UUID                UUID NOT NULL,
                ATTEMPTED_AT        TIMESTAMP WITH TIME ZONE NOT NULL,
                ERROR_MESSAGE       TEXT,
                OUTBOX_MESSAGE_UUID UUID NOT NULL,

                CONSTRAINT PK_OUTBOX_MESSAGE_ERROR PRIMARY KEY (UUID),
                CONSTRAINT FK_OUTBOX_MESSAGE_ERROR_OUTBOX_MESSAGE
                    FOREIGN KEY (OUTBOX_MESSAGE_UUID)
                    REFERENCES OUTBOX_MESSAGE(UUID)
                    ON DELETE CASCADE
            );
        </sql>
    </changeSet>

    <changeSet id="CONTROL-VEHICLE/create_outbox_message/30" author="OpenAI">
        <preConditions onFail="HALT" onError="HALT">
            <not>
                <indexExists indexName="idx_outbox_message_pending_next_attempt"/>
            </not>
        </preConditions>
        <sql>
            CREATE INDEX idx_outbox_message_pending_next_attempt
                ON outbox_message (next_attempt_at)
                WHERE status = 'PENDING';
        </sql>
    </changeSet>

    <changeSet id="CONTROL-VEHICLE/create_outbox_message/40" author="OpenAI">
        <preConditions onFail="HALT" onError="HALT">
            <not>
                <indexExists indexName="idx_outbox_message_in_progress_picked_next_attempt"/>
            </not>
        </preConditions>
        <sql>
            CREATE INDEX idx_outbox_message_in_progress_picked_next_attempt
                ON outbox_message (picked_at, next_attempt_at)
                WHERE status = 'IN_PROGRESS';
        </sql>
    </changeSet>

    <changeSet id="CONTROL-VEHICLE/create_outbox_message/50" author="OpenAI">
        <preConditions onFail="HALT" onError="HALT">
            <not>
                <indexExists indexName="idx_outbox_message_event_aggregate_created"/>
            </not>
        </preConditions>
        <sql>
            CREATE INDEX idx_outbox_message_event_aggregate_created
                ON outbox_message(event_type, aggregate_id, created_at);
        </sql>
    </changeSet>

</databaseChangeLog>


```
