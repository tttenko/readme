```java
 @Transactional
    public List<StsDataEntity> toApprove(List<UUID> uuids) {
        LinkedHashSet<UUID> requestedUuids = new LinkedHashSet<>(uuids);

        List<StsDataEntity> entities = stsDataRepository.findAllByUuidInAndDeleted(requestedUuids, false);

        validateAllFound(requestedUuids, entities);

        Map<UUID, Integer> order = new HashMap<>();
        int index = 0;
        for (UUID uuid : requestedUuids) {
            order.put(uuid, index++);
        }

        entities.sort(Comparator.comparingInt(entity -> order.get(entity.getUuid())));

        Map<UUID, StsStatus> oldStatuses = new HashMap<>();

        for (StsDataEntity entity : entities) {
            StsStatus oldStatus = entity.getStatusId();
            StsStatus newStatus = resolveToApproveStatus(oldStatus);

            oldStatuses.put(entity.getUuid(), oldStatus);
            entity.setStatusId(newStatus);
        }

        List<StsDataEntity> savedEntities = stsDataRepository.saveAll(entities);

        savedEntities.sort(Comparator.comparingInt(entity -> order.get(entity.getUuid())));

        for (StsDataEntity savedEntity : savedEntities) {
            StsStatus oldStatus = oldStatuses.get(savedEntity.getUuid());

            stsEventsHistoryOutboxService.sendToApproveEvent(savedEntity, oldStatus);
            stsTrackerHistoryOutboxService.sendToApproveStatus(savedEntity);
        }

        return savedEntities;
    }

     private void validateAllFound(Set<UUID> requestedUuids, List<StsDataEntity> entities) {
        Set<UUID> foundUuids = entities.stream()
                .map(StsDataEntity::getUuid)
                .collect(Collectors.toSet());

        List<UUID> missingUuids = requestedUuids.stream()
                .filter(uuid -> !foundUuids.contains(uuid))
                .toList();

        if (!missingUuids.isEmpty()) {
            throw new EntityNotFoundException("Записи СТС не найдены по uuid: " + missingUuids);
        }
    }

    private StsStatus resolveToApproveStatus(StsStatus currentStatus) {
        return switch (currentStatus) {
            case DRAFT -> StsStatus.TO_APPROVE_IN;
            case APPROVED -> StsStatus.TO_APPROVE_OUT;
            default -> throw new IllegalStateException(
                    "Операция toApprove недоступна для статуса: " + currentStatus.getTitle()
            );
        };
    }
```
