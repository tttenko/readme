```java
@Service
@RequiredArgsConstructor
public class StsStatusTransitionService {

    private final Scheme stsTrackerScheme;

    public StsStatus calculateNextStatus(StsDataEntity entity) {
        Node currentNode = stsTrackerScheme.getNode(entity.getStatusId().name());
        Node nextNode = stsTrackerScheme.calculateNext(currentNode, Map.of());

        return StsStatus.valueOf(nextNode.getId());
    }
}

@Service
@RequiredArgsConstructor
public class StsWorkflowService {

    private final StsDataService stsDataService;
    private final StsStatusTransitionService stsStatusTransitionService;
    private final StsEventsHistoryOutboxService stsEventsHistoryOutboxService;
    private final StsTrackerHistoryOutboxService stsTrackerHistoryOutboxService;

    @Transactional
    public StsDataEntity create(StsDataEntity entity) {
        entity.setStatusId(StsStatus.DRAFT);
        entity.setDeleted(false);

        StsDataEntity savedEntity = stsDataService.save(entity);

        stsEventsHistoryOutboxService.sendCreateEvent(savedEntity);
        stsTrackerHistoryOutboxService.sendCreatedStatus(savedEntity);

        return savedEntity;
    }

    @Transactional
    public List<StsDataEntity> toApprove(List<UUID> uuids) {
        List<StsDataEntity> entities = stsDataService.getByUuids(uuids);

        Map<UUID, StsStatus> oldStatuses = new HashMap<>();

        for (StsDataEntity entity : entities) {
            StsStatus oldStatus = entity.getStatusId();
            StsStatus newStatus = stsStatusTransitionService.calculateNextStatus(entity);

            oldStatuses.put(entity.getUuid(), oldStatus);
            entity.setStatusId(newStatus);
        }

        List<StsDataEntity> savedEntities = stsDataService.saveAll(entities);

        for (StsDataEntity savedEntity : savedEntities) {
            StsStatus oldStatus = oldStatuses.get(savedEntity.getUuid());

            stsEventsHistoryOutboxService.sendToApproveEvent(savedEntity, oldStatus);
            stsTrackerHistoryOutboxService.sendToApproveStatus(savedEntity);
        }

        return savedEntities;
    }
}


```
