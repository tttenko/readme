```java
@Service
@RequiredArgsConstructor
public class StsDataService {

    private final StsDataRepository stsDataRepository;

    @Transactional(readOnly = true)
    public Page<StsDataEntity> getPageByContractUuid(UUID contractUuid,
                                                     TopSkipOrderFilterParams parameters) {

        JpaAdapter adapter = new JpaAdapter(parameters);

        Specification<StsDataEntity> userSpecification = adapter.specification();

        Specification<StsDataEntity> specification = hasContractUuid(contractUuid)
                .and(isNotDeleted())
                .and(userSpecification);

        return stsDataRepository.findAll(specification, adapter.pageable());
    }

    @Transactional(readOnly = true)
    public StsDataEntity getByUuid(UUID uuid) {
        return stsDataRepository.findByUuidAndDeleted(uuid, false)
                .orElseThrow(() -> new EntityNotFoundException("Запись СТС не найдена по uuid: " + uuid));
    }

    @Transactional(readOnly = true)
    public List<StsDataEntity> getByUuids(List<UUID> uuids) {
        LinkedHashSet<UUID> requestedUuids = new LinkedHashSet<>(uuids);

        List<StsDataEntity> entities = stsDataRepository.findAllByUuidInAndDeleted(requestedUuids, false);

        validateAllFound(requestedUuids, entities);

        Map<UUID, Integer> order = new HashMap<>();
        int index = 0;
        for (UUID uuid : requestedUuids) {
            order.put(uuid, index++);
        }

        entities.sort(Comparator.comparingInt(entity -> order.get(entity.getUuid())));
        return entities;
    }

    @Transactional
    public StsDataEntity save(StsDataEntity entity) {
        return stsDataRepository.save(entity);
    }

    @Transactional
    public List<StsDataEntity> saveAll(List<StsDataEntity> entities) {
        return stsDataRepository.saveAll(entities);
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

    private Specification<StsDataEntity> hasContractUuid(UUID contractUuid) {
        return (root, query, criteriaBuilder) ->
                criteriaBuilder.equal(root.get("contractUuid"), contractUuid);
    }

    private Specification<StsDataEntity> isNotDeleted() {
        return (root, query, criteriaBuilder) ->
                criteriaBuilder.isFalse(root.get("deleted"));
    }
}
```
