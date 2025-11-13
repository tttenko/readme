```java

public class MdaCacheNotFoundException extends RuntimeException {
  private final String cacheName;

  public MdaCacheNotFoundException(String cacheName) {
    super("Cache not found: " + cacheName);
    this.cacheName = cacheName;
  }

  public String getCacheName() { return cacheName; }
}

error.cacheNotFound=Кэш недоступен или не сконфигурирован

@ExceptionHandler(MdaCacheNotFoundException.class)
@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
public ResponseEntity<Object> handleCacheNotFound(MdaCacheNotFoundException ex, WebRequest request) {
  log.error(ERROR_MESSAGE, ex);
  String message = getLocalizedErrorMessage("error.cacheNotFound");
  return createResponseEntity(message, new HttpHeaders(), HttpStatus.INTERNAL_SERVER_ERROR, request);
}

```
