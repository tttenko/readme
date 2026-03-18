```java

@Service
@RequiredArgsConstructor
public class StsDataService {

    private final StsDataRepository stsDataRepository;

    /**
     * Сохраняет запись СТС.
     */
    @Transactional
    public StsDataEntity create(StsDataEntity entity) {
        return stsDataRepository.save(entity);
    }

    /**
     * Возвращает страницу записей СТС, связанных с указанным договором и не помеченных как удалённые.
     */
    @Transactional(readOnly = true)
    public Page<StsDataEntity> getPageByContractUuid(UUID contractUuid,
                                                     Integer top,
                                                     Integer skip) {

        Pageable pageable = PageRequest.of(
                skip / top,
                top,
                Sort.by(Sort.Direction.DESC, "createdAt")
        );

        return stsDataRepository.findAllByContractUuidAndDeleted(contractUuid, false, pageable);
    }

    /**
     * Возвращает одну запись СТС по её уникальному идентификатору, если она существует и не удалена.
     */
    @Transactional(readOnly = true)
    public StsDataEntity getByUuid(UUID uuid) {
        return stsDataRepository.findByUuidAndDeleted(uuid, false)
                .orElseThrow(() -> new EntityNotFoundException("Запись СТС не найдена по uuid: " + uuid));
    }
}

@Component
@RequiredArgsConstructor
public class PageDtoMapper {

    private final StsDataMapper stsDataMapper;

    public PageDto<StsDataDto> toStsDataPageDto(Page<StsDataEntity> page) {
        PageDto<StsDataDto> result = new PageDto<>();
        result.setItems(page.getContent().stream().map(stsDataMapper::toDto).toList());
        result.setPage(page.getNumber());
        result.setSize(page.getSize());
        result.setTotalElements(page.getTotalElements());
        result.setTotalPages(page.getTotalPages());
        result.setFirst(page.isFirst());
        result.setLast(page.isLast());
        return result;
    }
}

@Component
@RequiredArgsConstructor
public class PageDtoMapper {

    private final StsDataMapper stsDataMapper;

    public PageDto<StsDataDto> toStsDataPageDto(Page<StsDataEntity> page) {
        PageDto<StsDataDto> result = new PageDto<>();
        result.setItems(page.getContent().stream().map(stsDataMapper::toDto).toList());
        result.setPage(page.getNumber());
        result.setSize(page.getSize());
        result.setTotalElements(page.getTotalElements());
        result.setTotalPages(page.getTotalPages());
        result.setFirst(page.isFirst());
        result.setLast(page.isLast());
        return result;
    }
}
```
