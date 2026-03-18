```java

@RestController
@RequiredArgsConstructor
public class StsDataControllerImpl implements StsDataController {

    private final StsDataService stsDataService;
    private final StsDataMapper stsDataMapper;
    private final StsDataPageMapper stsDataPageMapper;

    /**
     * Создаёт новую запись СТС на основе переданных данных.
     */
    @Override
    public ResultObj<StsDataDto> createStsData(CreateStsDataRequest request) {
        ZonedDateTime now = ZonedDateTime.now();

        StsDataEntity entity = stsDataMapper.toEntity(request);
        entity.setCreatedAt(now);
        entity.setUpdatedAt(now);
        entity.setUpdatedBy(request.getCreatedBy());
        entity.setDeleted(false);

        StsDataEntity savedEntity = stsDataService.create(entity);

        return getSuccessItemResponse(stsDataMapper.toDto(savedEntity));
    }

    /**
     * Возвращает страницу записей СТС, связанных с указанным договором.
     * Параметры $filter и $orderby пока принимаются только контрактом API и пока не обрабатываются.
     */
    @Override
    public ResultObj<PageDto<StsDataDto>> getStsDataByContractUuid(UUID contractUuid,
                                                                   String filter,
                                                                   String orderBy,
                                                                   Integer top,
                                                                   Integer skip) {
        Page<StsDataEntity> page = stsDataService.getPageByContractUuid(contractUuid, top, skip);

        return getSuccessPageResponse(stsDataPageMapper.toDto(page));
    }

    /**
     * Возвращает одну запись СТС по её уникальному идентификатору.
     */
    @Override
    public ResultObj<StsDataDto> getStsDataByUuid(UUID uuid) {
        StsDataEntity entity = stsDataService.getByUuid(uuid);

        return getSuccessItemResponse(stsDataMapper.toDto(entity));
    }
}
```
