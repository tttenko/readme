```java
public record StsBatchOperationError(
        UUID uuid,
        String code,
        String message
) {
}

public record StsBatchOperationResult<T>(
        List<T> processed,
        List<StsBatchOperationError> errors
) {
}

 @Getter
@Setter
@Schema(description = "Ошибка обработки записи СТС")
public class StsBatchOperationErrorDto {

    @Schema(description = "UUID записи СТС")
    private UUID uuid;

    @Schema(description = "Код ошибки")
    private String code;

    @Schema(description = "Описание ошибки")
    private String message;
}

@Getter
@Setter
@Schema(description = "Результат пакетной обработки СТС")
public class ToApproveStsDataResultDto {

    @Schema(description = "Успешно обработанные записи")
    private List<StsDataDto> processed;

    @Schema(description = "Ошибки по необработанным записям")
    private List<StsBatchOperationErrorDto> errors;
}

@Transactional(readOnly = true)
    public List<StsDataEntity> getExistingByUuids(List<UUID> uuids) {
        return stsDataRepository.findAllByUuidInAndDeleted(uuids, false);
    }

private static final String ERROR_NOT_FOUND = "STS_NOT_FOUND";
    private static final String ERROR_INVALID_STATUS = "INVALID_STATUS_TRANSITION";


 @Transactional
    public StsBatchOperationResult<StsDataEntity> toApprove(List<UUID> uuids) {

        LinkedHashSet<UUID> requestedUuids = new LinkedHashSet<>(uuids);

        List<StsDataEntity> existingEntities =
                stsDataService.getExistingByUuids(new ArrayList<>(requestedUuids));

        Map<UUID, StsDataEntity> entitiesByUuid = existingEntities.stream()
                .collect(Collectors.toMap(StsDataEntity::getUuid, Function.identity()));

        List<StsBatchOperationError> errors = new ArrayList<>();
        List<StsDataEntity> toSave = new ArrayList<>();
        Map<UUID, StsStatus> oldStatuses = new HashMap<>();

        for (UUID uuid : requestedUuids) {
            StsDataEntity entity = entitiesByUuid.get(uuid);

            if (entity == null) {
                errors.add(new StsBatchOperationError(
                        uuid,
                        ERROR_NOT_FOUND,
                        "Запись СТС не найдена"
                ));
                continue;
            }

            try {
                StsStatus oldStatus = entity.getStatusId();

                StsStatus newStatus =
                        stsStatusTransitionService.calculateNextStatus(entity, StsAction.TO_APPROVE);

                oldStatuses.put(entity.getUuid(), oldStatus);
                entity.setStatusId(newStatus);
                toSave.add(entity);

            } catch (Exception ex) {
                errors.add(new StsBatchOperationError(
                        uuid,
                        ERROR_INVALID_STATUS,
                        ex.getMessage()
                ));
            }
        }

        List<StsDataEntity> savedEntities = toSave.isEmpty()
                ? List.of()
                : stsDataService.saveAll(toSave);

        for (StsDataEntity savedEntity : savedEntities) {
            StsStatus oldStatus = oldStatuses.get(savedEntity.getUuid());

            stsEventsHistoryOutboxService.sendToApproveEvent(savedEntity, oldStatus);
            stsTrackerHistoryOutboxService.sendToApproveStatus(savedEntity);
        }

        return new StsBatchOperationResult<>(savedEntities, errors);
    }

@Override
public ResultObj<ToApproveStsDataResultDto> toApproveStsData(AuthorizedUser principal,
                                                             ToApproveStsDataRequest request) {

    StsBatchOperationResult<StsDataEntity> result =
            stsWorkFlowService.toApprove(request.getUuids());

    ToApproveStsDataResultDto response = new ToApproveStsDataResultDto();
    response.setProcessed(
            result.processed().stream()
                    .map(stsDataMapper::toDto)
                    .toList()
    );
    response.setErrors(
            result.errors().stream()
                    .map(this::toErrorDto)
                    .toList()
    );


    

```
