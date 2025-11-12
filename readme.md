```java

public class MdaInvalidCacheKeyException extends RuntimeException {
  private final String cacheName;
  private final Object item;

  public MdaInvalidCacheKeyException(String cacheName, Object item) {
    super("Invalid (null/blank) cache key. cache='" + cacheName + "', item=" + String.valueOf(item));
    this.cacheName = cacheName;
    this.item = item;
  }

  public String getCacheName() { return cacheName; }
  public Object getItem() { return item; }
}

@ExceptionHandler(MdaInvalidLoadedItemKeyException.class)
@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
public ResponseEntity<Object> handleInvalidLoadedItemKey(MdaInvalidLoadedItemKeyException ex, WebRequest req) {
  log.error(ERROR_MESSAGE, ex);
  return createResponseEntity(getLocalizedErrorMessage("error.invalidLoadedItemKey"),
                              new HttpHeaders(), HttpStatus.INTERNAL_SERVER_ERROR, req);
}
```
