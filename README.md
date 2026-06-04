```java

@Schema(description = "Запрос на создание enabler")
data class CreateEnablerRequest(

    @field:Schema(
        maxLength = 255,
        description = "Название enabler",
        required = true,
        nullable = false,
        pattern = ANY_CHAR_PATTERN,
        example = "Enabler 1"
    )
    val name: String,
)

@Schema(description = "Запрос на обновление enabler")
data class UpdateEnablerRequest(

    @field:Schema(
        maxLength = 255,
        description = "Название enabler",
        required = true,
        nullable = false,
        pattern = ANY_CHAR_PATTERN,
        example = "Enabler 1"
    )
    val name: String,

    @field:Schema(
        description = "Признак отключения enabler",
        required = true,
        nullable = false,
        example = "false"
    )
    val disabled: Boolean,
)

@Schema(description = "Enabler")
data class EnablerResponse(

    @Schema(
        maximum = "4503599627370496",
        minimum = "0",
        description = "Идентификатор enabler",
        required = true,
        nullable = false,
        pattern = "^\\d*$",
        example = "1"
    )
    val id: Long,

    @Schema(
        maxLength = 255,
        description = "Название enabler",
        required = true,
        nullable = false,
        pattern = ANY_CHAR_PATTERN,
        example = "Enabler 1"
    )
    val name: String? = null,

    @Schema(
        description = "Признак отключения enabler",
        required = true,
        nullable = false,
        example = "false"
    )
    val disabled: Boolean = false,
)

@Schema(description = "Ответ на создание enabler")
data class CreateEnablerResponse(

    @Schema(
        maximum = "4503599627370496",
        minimum = "0",
        description = "Идентификатор созданного enabler",
        required = true,
        nullable = false,
        pattern = "^\\d*$",
        example = "1"
    )
    val id: Long,
)

fun EnablerEntity.toEnablerResponse(): EnablerResponse {
    return EnablerResponse(
        id = id,
        name = name,
        disabled = disabled ?: false
    )
}

override fun findAll(): List<EnablerEntity>

    fun findAllByDisabledIsFalse(): List<EnablerEntity>

    fun findEnablerEntityById(id: Long): EnablerEntity?

    @Query(
        value = """
            select count(*) > 0
            from enabler e
            where lower(regexp_replace(e.name, '[[:space:]]+', '', 'g')) =
                  lower(regexp_replace(:name, '[[:space:]]+', '', 'g'))
        """,
        nativeQuery = true
    )
    fun existsByNormalizedName(
        @Param("name") name: String
    ): Boolean

    @Query(
        value = """
            select count(*) > 0
            from enabler e
            where e.id <> :id
              and lower(regexp_replace(e.name, '[[:space:]]+', '', 'g')) =
                  lower(regexp_replace(:name, '[[:space:]]+', '', 'g'))
        """,
        nativeQuery = true
    )
    fun existsByNormalizedNameAndIdNot(
        @Param("name") name: String,
        @Param("id") id: Long
    ): Boolean

@Service
open class EnablerService(
    private val enablerRepository: EnablerRepository
) {

    private val log by logger()

    @Transactional(readOnly = true)
    open fun getEnablerReferences(includeDisabled: Boolean): List<EnablerResponse> {
        return if (includeDisabled) {
            enablerRepository.findAll()
        } else {
            enablerRepository.findAllByDisabledIsFalse()
        }.map { it.toEnablerResponse() }
    }

    @Transactional
    open fun createEnabler(request: CreateEnablerRequest): CreateEnablerResponse {
        validateNameNotExists(
            name = request.name,
            operationDetails = "Добавление enabler"
        )

        val entity = EnablerEntity().apply {
            name = request.name
            disabled = false
        }

        val savedEntity = enablerRepository.save(entity)

        return CreateEnablerResponse(
            id = savedEntity.id
        )
    }

    @Transactional
    open fun updateEnabler(id: Long, request: UpdateEnablerRequest) {
        val entity = enablerRepository.findEnablerEntityById(id)
            ?: run {
                log.logWarning(
                    operationDetails = "Обновление enabler"
                )

                throw ResponseStatusException(
                    HttpStatus.NOT_FOUND,
                    "Enabler с id $id не найден"
                )
            }

        validateNameNotExistsForUpdate(
            id = id,
            name = request.name,
            operationDetails = "Обновление enabler"
        )

        entity.name = request.name
        entity.disabled = request.disabled

        enablerRepository.save(entity)
    }

    private fun validateNameNotExists(
        name: String,
        operationDetails: String
    ) {
        if (enablerRepository.existsByNormalizedName(name)) {
            val errorMessage = "Название $name уже существует"

            log.logError(
                operationDetails = operationDetails,
                errorMessage = errorMessage
            )

            throw AiBadRequestException(
                errorCode = "ENABLER_NAME_ALREADY_EXISTS",
                message = errorMessage,
                operationDetails = operationDetails
            )
        }
    }

    private fun validateNameNotExistsForUpdate(
        id: Long,
        name: String,
        operationDetails: String
    ) {
        if (enablerRepository.existsByNormalizedNameAndIdNot(name, id)) {
            val errorMessage = "Название $name уже существует"

            log.logError(
                operationDetails = operationDetails,
                errorMessage = errorMessage
            )

            throw AiBadRequestException(
                errorCode = "ENABLER_NAME_ALREADY_EXISTS",
                message = errorMessage,
                operationDetails = operationDetails
            )
        }
    }
}

@Validated
@RestController
@RequestMapping("/api/ai/v1/reference/enabler")
@Tag(description = "Enabler", name = "aigw enabler references")
open class EnablerController(
    private val enablerService: EnablerService
) {

    @GetMapping
    @Operation(
        summary = "Получение списка enabler",
        responses = [
            ApiResponse(
                responseCode = "200",
                content = [
                    Content(
                        mediaType = "application/json",
                        schema = Schema(implementation = EnablerResponse::class)
                    )
                ]
            )
        ]
    )
    @ExceptionApiResponses
    @PreAuthorize("hasAnyAuthority('CMS_ADMIN', 'PROJECT_OFFICE')")
    open fun getEnablerReferences(
        @RequestParam(defaultValue = "false")
        includeDisabled: Boolean
    ): List<EnablerResponse> {
        return enablerService.getEnablerReferences(includeDisabled)
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(
        summary = "Создание enabler",
        responses = [
            ApiResponse(
                responseCode = "201",
                content = [
                    Content(
                        mediaType = "application/json",
                        schema = Schema(implementation = CreateEnablerResponse::class)
                    )
                ]
            )
        ]
    )
    @ExceptionApiResponses
    @PreAuthorize("hasAuthority('CMS_ADMIN')")
    open fun createEnabler(
        @Valid @RequestBody request: CreateEnablerRequest
    ): CreateEnablerResponse {
        return enablerService.createEnabler(request)
    }

    @PutMapping("/{id}")
    @Operation(
        summary = "Обновление enabler",
        responses = [
            ApiResponse(
                responseCode = "200"
            )
        ]
    )
    @ExceptionApiResponses
    @PreAuthorize("hasAuthority('CMS_ADMIN')")
    open fun updateEnabler(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateEnablerRequest
    ) {
        enablerService.updateEnabler(
            id = id,
            request = request
        )
    }
}
```
