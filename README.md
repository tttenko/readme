```java

@Valid
@Validated
@RequestMapping(value = "/api/v1/sts", produces = APPLICATION_JSON_VALUE)
@Tag(
        name = "Sts data controller",
        description = "Сервис управления данными СТС"
)
public interface StsDataController {

    @PostMapping
    @Operation(
            operationId = "createStsData",
            summary = "Создание записи СТС"
    )
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "Запись СТС успешно создана"),
            @ApiResponse(responseCode = "400", description = "Некорректные параметры запроса"),
            @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<StsDataDto>> createStsData(
            @Valid @RequestBody CreateStsDataRequest request);

    @GetMapping
    @Operation(
            operationId = "getStsDataByContractUuid",
            summary = "Получение списка СТС по contractUuid"
    )
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "Список СТС успешно получен"),
            @ApiResponse(responseCode = "400", description = "Некорректные параметры запроса"),
            @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<StsDataDto>> getStsDataByContractUuid(
            @RequestParam(name = "contractUuid")
            @NotNull(message = "Параметр contractUuid не должен быть null")
            UUID contractUuid);

    @GetMapping(value = "/{uuid}")
    @Operation(
            operationId = "getStsDataByUuid",
            summary = "Получение записи СТС по uuid"
    )
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "Запись СТС успешно получена"),
            @ApiResponse(responseCode = "400", description = "Некорректный uuid"),
            @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<StsDataDto>> getStsDataByUuid(
            @PathVariable("uuid")
            @NotNull(message = "Параметр uuid не должен быть null")
            UUID uuid);
}

@Valid
@Validated
@RequestMapping(value = "/api/v1/sts", produces = APPLICATION_JSON_VALUE)
@Tag(
        name = "Sts data controller",
        description = "Сервис управления данными СТС"
)
public interface StsDataController {

    @PostMapping
    @Operation(
            operationId = "createStsData",
            summary = "Создание записи СТС"
    )
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "Запись СТС успешно создана"),
            @ApiResponse(responseCode = "400", description = "Некорректные параметры запроса"),
            @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<StsDataDto>> createStsData(
            @Valid @RequestBody CreateStsDataRequest request);

    @GetMapping
    @Operation(
            operationId = "getStsDataByContractUuid",
            summary = "Получение списка СТС по contractUuid"
    )
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "Список СТС успешно получен"),
            @ApiResponse(responseCode = "400", description = "Некорректные параметры запроса"),
            @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<StsDataDto>> getStsDataByContractUuid(
            @RequestParam(name = "contractUuid")
            @NotNull(message = "Параметр contractUuid не должен быть null")
            UUID contractUuid);

    @GetMapping(value = "/{uuid}")
    @Operation(
            operationId = "getStsDataByUuid",
            summary = "Получение записи СТС по uuid"
    )
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "Запись СТС успешно получена"),
            @ApiResponse(responseCode = "400", description = "Некорректный uuid"),
            @ApiResponse(responseCode = "500", description = "Внутренняя ошибка сервиса")
    })
    ResultObj<List<StsDataDto>> getStsDataByUuid(
            @PathVariable("uuid")
            @NotNull(message = "Параметр uuid не должен быть null")
            UUID uuid);
}

@Slf4j
@Validated
@RestController
@RequiredArgsConstructor
public class StsDataControllerImpl implements StsDataController {

    private final StsDataService stsDataService;
    private final ResponseHandler responseHandler;

    @Override
    public ResultObj<List<StsDataDto>> createStsData(@Valid @RequestBody CreateStsDataRequest request) {
        return responseHandler.executeOrThrow(() ->
                getSuccessResponse(List.of(stsDataService.create(request))));
    }

    @Override
    public ResultObj<List<StsDataDto>> getStsDataByContractUuid(
            @RequestParam(name = "contractUuid")
            @NotNull(message = "Параметр contractUuid не должен быть null")
            UUID contractUuid) {

        return responseHandler.executeOrThrow(() ->
                getSuccessResponse(stsDataService.getByContractUuid(contractUuid)));
    }

    @Override
    public ResultObj<List<StsDataDto>> getStsDataByUuid(
            @PathVariable("uuid")
            @NotNull(message = "Параметр uuid не должен быть null")
            UUID uuid) {

        return responseHandler.executeOrThrow(() ->
                getSuccessResponse(List.of(stsDataService.getByUuid(uuid))));
    }
}

public interface StsDataService {

    StsDataDto create(CreateStsDataRequest request);

    List<StsDataDto> getByContractUuid(UUID contractUuid);

    StsDataDto getByUuid(UUID uuid);
}

@Slf4j
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class StsDataServiceImpl implements StsDataService {

    private final StsDataRepository stsDataRepository;
    private final StsDataMapper stsDataMapper;

    @Override
    @Transactional
    public StsDataDto create(CreateStsDataRequest request) {
        StsDataEntity entity = stsDataMapper.toEntity(request);
        StsDataEntity savedEntity = stsDataRepository.save(entity);
        return stsDataMapper.toDto(savedEntity);
    }

    @Override
    public List<StsDataDto> getByContractUuid(UUID contractUuid) {
        return stsDataRepository.findAllByContractUuidAndDeletedFalseOrderByCreatedAtDesc(contractUuid)
                .stream()
                .map(stsDataMapper::toDto)
                .toList();
    }

    @Override
    public StsDataDto getByUuid(UUID uuid) {
        StsDataEntity entity = stsDataRepository.findByUuidAndDeletedFalse(uuid)
                .orElseThrow(() -> new EntityNotFoundException("Запись СТС не найдена по uuid: " + uuid));

        return stsDataMapper.toDto(entity);
    }
}

public interface StsDataRepository extends JpaRepository<StsDataEntity, UUID> {

    List<StsDataEntity> findAllByContractUuidAndDeletedFalseOrderByCreatedAtDesc(UUID contractUuid);

    Optional<StsDataEntity> findByUuidAndDeletedFalse(UUID uuid);
}

@Entity
@Setter
@Getter
@Table(name = "sts_data")
public class StsDataEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(name = "uuid", nullable = false, updatable = false)
    private UUID uuid;

    @Column(name = "contract_uuid", nullable = false)
    private UUID contractUuid;

    @Column(name = "tb_id", nullable = false, length = 4)
    private String tbId;

    @Column(name = "vehicle_number", nullable = false, length = 15)
    private String vehicleNumber;

    @Column(name = "vehicle_brand", nullable = false, length = 35)
    private String vehicleBrand;

    @Column(name = "comment", length = 128)
    private String comment;

    @Column(name = "status_id", nullable = false, length = 2)
    private String statusId;

    @Column(name = "created_by", nullable = false, length = 100)
    private String createdBy;

    @Column(name = "changed_by", nullable = false, length = 100)
    private String changedBy;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;

    @Column(name = "changed_at", nullable = false)
    private LocalDateTime changedAt;

    @Column(name = "is_deleted", nullable = false)
    private boolean deleted;

    @PrePersist
    public void prePersist() {
        LocalDateTime now = LocalDateTime.now();

        if (createdAt == null) {
            createdAt = now;
        }
        if (changedAt == null) {
            changedAt = now;
        }
        if (changedBy == null || changedBy.isBlank()) {
            changedBy = createdBy;
        }
    }

    @PreUpdate
    public void preUpdate() {
        changedAt = LocalDateTime.now();
    }

    @Override
    public final boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || Hibernate.getClass(this) != Hibernate.getClass(o)) {
            return false;
        }
        StsDataEntity other = (StsDataEntity) o;
        return uuid != null && Objects.equals(uuid, other.uuid);
    }

    @Override
    public final int hashCode() {
        return Hibernate.getClass(this).hashCode();
    }
}

@Getter
@Setter
@Schema(description = "Запрос на создание записи СТС")
public class CreateStsDataRequest {

    @NotNull(message = "contractUuid не должен быть null")
    @Schema(description = "Uuid договора", requiredMode = Schema.RequiredMode.REQUIRED)
    private UUID contractUuid;

    @NotBlank(message = "tbId не должен быть пустым")
    @Size(max = 4, message = "tbId не должен превышать 4 символа")
    @Schema(description = "Код ТБ", requiredMode = Schema.RequiredMode.REQUIRED, example = "1234")
    private String tbId;

    @NotBlank(message = "vehicleNumber не должен быть пустым")
    @Size(max = 15, message = "vehicleNumber не должен превышать 15 символов")
    @Schema(description = "Номер ТС", requiredMode = Schema.RequiredMode.REQUIRED, example = "А123АА777")
    private String vehicleNumber;

    @NotBlank(message = "vehicleBrand не должен быть пустым")
    @Size(max = 35, message = "vehicleBrand не должен превышать 35 символов")
    @Schema(description = "Марка ТС", requiredMode = Schema.RequiredMode.REQUIRED, example = "КамАЗ")
    private String vehicleBrand;

    @Size(max = 128, message = "comment не должен превышать 128 символов")
    @Schema(description = "Примечание", example = "Тестовая запись")
    private String comment;

    @NotBlank(message = "statusId не должен быть пустым")
    @Size(max = 2, message = "statusId не должен превышать 2 символа")
    @Schema(description = "Код статуса", requiredMode = Schema.RequiredMode.REQUIRED, example = "01")
    private String statusId;

    @NotBlank(message = "createdBy не должен быть пустым")
    @Size(max = 100, message = "createdBy не должен превышать 100 символов")
    @Schema(description = "Код автора создания", requiredMode = Schema.RequiredMode.REQUIRED, example = "supplier_user")
    private String createdBy;
}

@Component
public class StsDataMapper {

    public StsDataEntity toEntity(CreateStsDataRequest request) {
        StsDataEntity entity = new StsDataEntity();
        entity.setContractUuid(request.getContractUuid());
        entity.setTbId(request.getTbId());
        entity.setVehicleNumber(request.getVehicleNumber());
        entity.setVehicleBrand(request.getVehicleBrand());
        entity.setComment(request.getComment());
        entity.setStatusId(request.getStatusId());
        entity.setCreatedBy(request.getCreatedBy());
        entity.setChangedBy(request.getCreatedBy());
        entity.setDeleted(false);
        return entity;
    }

    public StsDataDto toDto(StsDataEntity entity) {
        StsDataDto dto = new StsDataDto();
        dto.setUuid(entity.getUuid());
        dto.setContractUuid(entity.getContractUuid());
        dto.setTbId(entity.getTbId());
        dto.setVehicleNumber(entity.getVehicleNumber());
        dto.setVehicleBrand(entity.getVehicleBrand());
        dto.setComment(entity.getComment());
        dto.setStatusId(entity.getStatusId());
        dto.setCreatedBy(entity.getCreatedBy());
        dto.setChangedBy(entity.getChangedBy());
        dto.setCreatedAt(entity.getCreatedAt());
        dto.setChangedAt(entity.getChangedAt());
        dto.setDeleted(entity.isDeleted());
        return dto;
    }
}



```
