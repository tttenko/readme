```java

@ExceptionHandler(MissingCalendarDataException.class)
@ResponseStatus(HttpStatus.NOT_FOUND)
public ResponseEntity<Object> handleMissingCalendarData(
        MissingCalendarDataException ex,
        WebRequest request
) {
    log.error(ERROR_MESSAGE, ex);

    return createResponseEntity(
            // если getLocalizedErrorMessage поддерживает параметры:
            getLocalizedErrorMessage("error.calendarDataMissing", ex.getDateShort()),
            new HttpHeaders(),
            HttpStatus.NOT_FOUND,
            request
    );
}
```
