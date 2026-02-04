```java

public class MdContractViolationException extends RuntimeException {

    private final String messageCode;

    public MdContractViolationException(String messageCode, String message) {
        super(message);
        this.messageCode = messageCode;
    }

    public MdContractViolationException(String messageCode, String message, Throwable cause) {
        super(message, cause);
        this.messageCode = messageCode;
    }

    public String getMessageCode() {
        return messageCode;
    }
}

@ExceptionHandler(MdContractViolationException.class)
@ResponseStatus(HttpStatus.BAD_GATEWAY) // 502 — внешний сервис вернул некорректный ответ
public ResponseEntity<Object> handleMdContractViolation(
        MdContractViolationException ex,
        WebRequest request
) {
    log.error(ERROR_MESSAGE, ex);

    // если используешь версию с messageCode:
    String message = getLocalizedErrorMessage(ex.getMessageCode());

    // если используешь простую версию без кода — тогда:
    // String message = getLocalizedErrorMessage("error.mdInvalidResponse");

    return createResponseEntity(
            message,
            new HttpHeaders(),
            HttpStatus.BAD_GATEWAY,
            request
    );
}

error.mdInvalidResponse=ЦС МД вернул некорректные данные: {0}
```
