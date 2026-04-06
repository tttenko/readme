```java
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

```
