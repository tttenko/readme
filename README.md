```java

@Slf4j
@RequiredArgsConstructor
@RestControllerAdvice(assignableTypes = {
        StsDataControllerImpl.class
})
public class GlobalExceptionHandler extends ResponseExceptionHandler {

    private static final String ERROR_MESSAGE =
            "RestErrorHandler: Error occurred while processing request:";

    /**
     * Обрабатывает исключение ValidationException.
     */
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ResponseEntity<Object> handleBadRequestException(
            ConstraintViolationException ex,
            WebRequest request
    ) {
        log.error(ERROR_MESSAGE, ex);

        String validationMessage = ex.getConstraintViolations().stream()
                .findFirst()
                .map(ConstraintViolation::getMessage)
                .orElse("");

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

    /**
     * Обрабатывает случай, когда запись СТС не найдена.
     */
    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ResponseEntity<Object> handleEntityNotFoundException(
            EntityNotFoundException ex,
            WebRequest request
    ) {
        log.error(ERROR_MESSAGE, ex);

        return createResponseEntity(
                ex.getMessage(),
                new HttpHeaders(),
                HttpStatus.NOT_FOUND,
                request
        );
    }

    private String getLocalizedErrorMessage(String code, Object... args) {
        return Objects.requireNonNull(getMessageSource())
                .getMessage(code, args, Locale.getDefault());
    }
}

error.notFoundData=Запрашиваемые данные не найдены
```
