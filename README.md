```java
 /**
     * TODO Временная авторизация:
     * доступ на отправку СТС на согласование проверяется по principal.getUserType().
     * После появления корректных permission / ABAC-проверок заменить на полноценную авторизацию по правам.
     */
    @PatchMapping("/sts/to_approve")
    @PreAuthorize("#principal.userType.name() == 'SUPPLIER'")
    @Operation(
            operationId = "toApproveStsData",
            summary = "Отправка СТС на согласование"
    )
    @ApiResponse(responseCode = "200", description = "СТС успешно отправлены на согласование")
    @ApiResponse(responseCode = "400", description = "Некорректные параметры запроса или недопустимый статус")
    @ApiResponse(responseCode = "403", description = "Операция доступна только поставщику")
    @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    ResultObj<List<StsDataDto>> toApproveStsData(
            AuthorizedUser principal,
            @Valid @RequestBody ToApproveStsDataRequest request
    );

@Override
    public ResultObj<List<StsDataDto>> toApproveStsData(AuthorizedUser principal,
                                                        ToApproveStsDataRequest request) {
        List<StsDataEntity> updatedEntities = stsDataService.toApprove(request.getUuids());

        List<StsDataDto> dtos = updatedEntities.stream()
                .map(stsDataMapper::toDto)
                .toList();

        return ResultObj.from(dtos);
    }

    @Getter
@Setter
@Schema(description = "Запрос на отправку СТС на согласование")
public class ToApproveStsDataRequest {

    @NotEmpty(message = "Список uuid не должен быть пустым")
    @Schema(description = "Список uuid записей СТС", requiredMode = Schema.RequiredMode.REQUIRED)
    private List<@NotNull(message = "uuid не должен быть null") UUID> uuids;
}

public void sendToApproveEvent(StsDataEntity entity, StsStatus oldStatus) {
        EventCreateDto payload = EventCreateDto.builder()
                .serviceId(StsHistoryEventIds.SERVICE_ID)
                .id(StsHistoryEventIds.STS_UPDATE)
                .entityUuid(entity.getUuid())
                .submittedAt(entity.getUpdatedAt())
                .submittedBy(entity.getUpdatedBy().toString())
                .session(NO_SESSION)
                .userName(resolveFullName())
                .userNode(NO_USER_NODE)
                .parameters(List.of(
                        parameter("vehicle_num", entity.getVehicleNumber()),
                        parameter("vehicle_name", entity.getVehicleBrand())
                ))
                .changedFields(List.of(
                        changedField("status", oldStatus.getTitle(), entity.getStatusId().getTitle())
                ))
                .build();

        publishEvent(entity.getUpdatedBy(), entity.getUuid().toString(), payload);
    }

    private void publishEvent(UUID createdBy, String aggregateId, EventCreateDto payload) {
        OutboxMessage message = outboxMessageService.addOutboxMessage(
                OutboxMessageEventType.CREATE_EVENT,
                createdBy,
                aggregateId,
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

    private EventChangedFieldCreateDto changedField(String id, String oldValue, String newValue) {
        return EventChangedFieldCreateDto.builder()
                .id(id)
                .oldValue(oldValue)
                .newValue(newValue)
                .build();
    }

public void sendToApproveStatus(StsDataEntity entity) {
        HistoryNewDto payload = HistoryNewDto.builder()
                .code(StsTrackerSchemeCodes.STATUS_STS)
                .entityUuid(entity.getUuid())
                .operation(StsAction.TO_APPROVE.getId())
                .status(entity.getStatusId().name())
                .comment(null)
                .createdBy(entity.getUpdatedBy())
                .extCreatedBy(entity.getUpdatedBy().toString())
                .userName(entity.getUpdatedBy().toString())
                .userPosition(null)
                .build();

        publishTrackerMessage(entity.getUpdatedBy(), entity.getUuid().toString(), payload);
    }

    private void publishTrackerMessage(UUID createdBy, String aggregateId, HistoryNewDto payload) {
        OutboxMessage message = outboxMessageService.addOutboxMessage(
                OutboxMessageEventType.SEND_TRACKER_HISTORY,
                createdBy,
                aggregateId,
                payload
        );

        applicationEventPublisher.publishEvent(new OutboxMessageEvent(message.getUuid()));
    }


```
