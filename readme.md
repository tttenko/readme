```java
/**
 * Исключение, выбрасываемое при некорректном запросе к адаптеру МД.
 *
 * Используется для случаев, когда входные параметры API
 * не соответствуют требованиям спецификации.
 */
public class MdaIncorrectRequestException extends RuntimeException {

    /**
     * Создаёт новое исключение с указанием причины.
     *
     * @param message сообщение с описанием ошибки
     */
    public MdaIncorrectRequestException(String message) {
        super(message);
    }

    /**
     * Создаёт новое исключение без сообщения.
     */
    public MdaIncorrectRequestException() {
        super();
    }
}


@ExceptionHandler(MdaIncorrectRequestException.class)
@ResponseStatus(value = HttpStatus.BAD_REQUEST)
public ResponseEntity<Object> handleMdaIncorrectRequestException(Exception ex, WebRequest request) {
    log.error(ERROR_MESSAGE, ex);

    return createResponseEntity(
            getLocalizedErrorMessage("error.incorrectRequest"),
            new HttpHeaders(),
            HttpStatus.BAD_REQUEST,
            request
    );
}
```
