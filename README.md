```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class StsHistoryEventIds {

    public static final String SERVICE_ID = "CI07150460_csportalControlVehcl";

    public static final String STS_CREATE = "STS_CREATE";
    public static final String STS_UPDATE = "STS_UPDATE";
    public static final String STS_STATUS_CHANGE = "STS_STATUS_CHANGE";
    public static final String STS_DELETE = "STS_DELETE";
}

@Getter
@RequiredArgsConstructor
public enum StsAction {

    TO_APPROVE("toApprove", "Индикатор отправки на согласование включения СТС"),
    DELETE("delete", "Индикатор удаления СТС"),
    CREATE_STS("createSts", "Индикатор создания СТС"),
    APPROVE("approve", "Индикатор согласования СТС"),
    REJECT("reject", "Индикатор отклонения СТС"),
    UPDATE("update", "Индикатор изменения СТС");

    private final String id;
    private final String description;
}

@Getter
@RequiredArgsConstructor
public enum StsStatus {

    DRAFT("01", "Черновик"),
    TO_APPROVE_IN("02", "На согласовании включения"),
    TO_APPROVE_OUT("03", "На согласовании удаления"),
    APPROVED("04", "Включен в договор"),
    DELETED("05", "Удален");

    private final String id;
    private final String title;

    public static StsStatus byId(String id) {
        return Arrays.stream(values())
                .filter(status -> Objects.equals(status.id, id))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("Неизвестный статус СТС: " + id));
    }

    public static String titleById(String id) {
        return byId(id).getTitle();
    }
}

@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class StsHistoryEventIds {

    public static final String SERVICE_ID = "CI07150460_csportalControlVehcl";

    public static final String STS_CREATE = "STS_CREATE";
    public static final String STS_UPDATE = "STS_UPDATE";
    public static final String STS_STATUS_CHANGE = "STS_STATUS_CHANGE";
    public static final String STS_DELETE = "STS_DELETE";
}

@Transactional
    public StsDataEntity create(StsDataEntity entity) {
        entity.setStatusId(StsStatus.DRAFT.getId());
        entity.setDeleted(false);

        StsDataEntity savedEntity = stsDataRepository.save(entity);

        stsEventsHistoryService.sendCreateEvent(savedEntity);

        return savedEntity;
    }

@Slf4j
@RequiredArgsConstructor
@RestControllerAdvice(assignableTypes = {
        StsDataControllerImpl.class
})
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ResponseEntity<Object> handleBadRequestException(
            ConstraintViolationException ex,
            WebRequest request) {

        String validationMessage = ex.getConstraintViolations().stream()
                .findFirst()
                .map(ConstraintViolation::getMessage)
                .orElse("");

        log.error("Request validation failed: {}", validationMessage);

        String errorMessage = getLocalizedErrorMessage(
                "error.incorrectRequest",
                validationMessage
        );

        return createResponseEntity(
                errorMessage,
                new HttpHeaders(),
                HttpStatus.BAD_REQUEST,
                request
        );
    }

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers,
            HttpStatusCode status,
            WebRequest request) {

        String validationMessage = ex.getBindingResult().getFieldErrors().stream()
                .findFirst()
                .map(DefaultMessageSourceResolvable::getDefaultMessage)
                .orElse("Некорректные параметры запроса");

        log.error("Request body validation failed: {}", validationMessage);

        return createResponseEntity(
                validationMessage,
                headers,
                HttpStatus.BAD_REQUEST,
                request
        );
    }

    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ResponseEntity<Object> handleEntityNotFoundException(
            EntityNotFoundException ex,
            WebRequest request) {

        log.error("Entity not found: {}", ex.getMessage());

        return createResponseEntity(
                ex.getMessage(),
                new HttpHeaders(),
                HttpStatus.NOT_FOUND,
                request
        );
    }

    @ExceptionHandler(HistoryIntegrationException.class)
    @ResponseStatus(HttpStatus.BAD_GATEWAY)
    public ResponseEntity<Object> handleHistoryIntegrationException(
            HistoryIntegrationException ex,
            WebRequest request) {

        log.error("Events history integration failed: {}", ex.getMessage(), ex);

        return createResponseEntity(
                ex.getMessage(),
                new HttpHeaders(),
                HttpStatus.BAD_GATEWAY,
                request
        );
    }

    private String getLocalizedErrorMessage(String code, Object... args) {
        return Objects.requireNonNull(getMessageSource())
                .getMessage(code, args, Locale.getDefault());
    }
}


```
