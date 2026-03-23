```java
@Service
@RequiredArgsConstructor
public class StsEventsHistoryService {

    private final EventsHistoryClient eventsHistoryClient;

    public void sendCreateEvent(StsDataEntity entity) {
        EventCreateDto event = EventCreateDto.builder()
                .serviceId(StsHistoryEventIds.SERVICE_ID)
                .id(StsHistoryEventIds.STS_CREATE)
                .entityUuid(entity.getUuid())
                .submittedAt(entity.getCreatedAt())
                .submittedBy(defaultValue(entity.getCreatedBy()))
                .session("NO-SESSION")
                .userName(defaultValue(entity.getCreatedBy()))
                .userNode("NO-USERNODE")
                .parameters(List.of(
                        parameter("sts_uuid", entity.getUuid().toString()),
                        parameter("contract_uuid", entity.getContractUuid().toString()),
                        parameter("tb_id", entity.getTbCode()),
                        parameter("vehicle_num", entity.getVehicleNumber()),
                        parameter("vehicle_name", entity.getVehicleBrand()),
                        parameter("comment", defaultValue(entity.getComment())),
                        parameter("status", StsStatus.titleById(entity.getStatusId()))
                ))
                .build();

        try {
            eventsHistoryClient.createEvent(event);
        } catch (Exception ex) {
            throw new HistoryIntegrationException("Не удалось записать событие в историю операций", ex);
        }
    }

    private EventParameterDto parameter(String id, String value) {
        return EventParameterDto.builder()
                .id(id)
                .value(defaultValue(value))
                .build();
    }

    private String defaultValue(String value) {
        return value == null ? "" : value;
    }
}
```
