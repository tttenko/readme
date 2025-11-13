```java

/**
 * Обработчик ошибок пакетной загрузки МДА-данных.
 */
@ExceptionHandler(MdaBatchLoadException.class)
@ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)
public ResponseEntity<Object> handleMdaBatchLoadException(MdaBatchLoadException ex, WebRequest request) {
    ...
}

/**
 * Обработчик ошибок с некорректным ключом кеша при загрузке МДА-данных.
 */
@ExceptionHandler(MdaInvalidCacheKeyException.class)
@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
public ResponseEntity<Object> handleInvalidLoadedItemKey(MdaInvalidCacheKeyException ex, WebRequest request) {
    ...
}

/**
 * Обработчик случая, когда запрашиваемый кеш МДА не найден.
 */
@ExceptionHandler(MdaCacheNotFoundException.class)
@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
public ResponseEntity<Object> handleCacheNotFound(MdaCacheNotFoundException ex, WebRequest request) {
    ...
}
MdaBatchLoadException
java
Копировать код
/**
 * Исключение, сигнализирующее об ошибке пакетной загрузки данных МДА.
 */
public class MdaBatchLoadException extends RuntimeException {

    /**
     * Создаёт исключение с причиной ошибки загрузки.
     *
     * @param cause причина ошибки
     */
    public MdaBatchLoadException(Throwable cause) {
        super("Load data failed", cause);
    }

    /**
     * Создаёт исключение с заданным сообщением.
     *
     * @param message текст сообщения об ошибке
     */
    public MdaBatchLoadException(String message) {
        super(message);
    }
}
MdaCacheNotFoundException
java
Копировать код
/**
 * Исключение, выбрасываемое при отсутствии указанного кеша МДА.
 */
@Getter
public class MdaCacheNotFoundException extends RuntimeException {

    private final String cacheName;

    /**
     * Создаёт исключение для не найденного кеша.
     *
     * @param cacheName имя кеша
     */
    public MdaCacheNotFoundException(String cacheName) {
        super("Cache not found: " + cacheName);
        this.cacheName = cacheName;
    }
}
MdaInvalidCacheKeyException
java
Копировать код
/**
 * Исключение, обозначающее неверный (null/пустой) ключ кеша МДА.
 */
@Getter
public class MdaInvalidCacheKeyException extends RuntimeException {

    private final String cacheName;
    private final Object item;

    /**
     * Создаёт исключение для некорректного ключа кеша и элемента.
     *
     * @param cacheName имя кеша
     * @param item      объект, для которого сформирован неверный ключ
     */
    public MdaInvalidCacheKeyException(String cacheName, Object item) {
        super("Invalid (null/blank) cache key. cache='" + cacheName + "', item=" + String.valueOf(item));
        this.cacheName = cacheName;
        this.item = item;
    }
}
LoaderTerBankByCode
java
Копировать код
/**
 * Batch-загрузчик тербанков по коду без реквизитов.
 */
@Component
@RequiredArgsConstructor
public class LoaderTerBankByCode implements BatchLoader<TerBankDto> {

    /**
     * Возвращает имя кеша для тербанков по коду.
     *
     * @return имя кеша
     */
    @Override
    public String cacheName() {
        return TerBankService2.TB_BY_CODE;
    }

    /**
     * Возвращает тип элементов, загружаемых данным лоадером.
     *
     * @return класс DTO тербанка
     */
    @Override
    public Class<TerBankDto> elementType() {
        return TerBankDto.class;
    }

    /**
     * Извлекает ключ кеша из DTO тербанка.
     *
     * @param v DTO тербанка
     * @return код тербанка
     */
    @Override
    public String extractKey(TerBankDto v) {
        return v.getTbCode();
    }

    /**
     * Загружает список тербанков по указанным кодам.
     *
     * @param codes список кодов тербанков
     * @return загруженные DTO тербанков
     */
    @Override
    public List<TerBankDto> fetchByKeys(List<String> codes) {
        var resp = master.requestData(
            props.getSlugValueForTerBank(),
            codes,
            SearchRequestProperties.Context.BOOK
        );
        return createResult(resp, mapper);
    }
}
LoaderTerBankWithRequisiteByCode
java
Копировать код
/**
 * Batch-загрузчик тербанков по коду вместе с реквизитами.
 */
@Component
@RequiredArgsConstructor
public class LoaderTerBankWithRequisiteByCode implements BatchLoader<TerBankWithRequisiteDto> {

    /**
     * Возвращает имя кеша для тербанков с реквизитами по коду.
     *
     * @return имя кеша
     */
    @Override
    public String cacheName() {
        return TerBankService2.TB_REQ_BY_CODE;
    }

    /**
     * Возвращает тип элементов, загружаемых данным лоадером.
     *
     * @return класс DTO тербанка с реквизитами
     */
    @Override
    public Class<TerBankWithRequisiteDto> elementType() {
        return TerBankWithRequisiteDto.class;
    }

    /**
     * Извлекает ключ кеша из DTO тербанка с реквизитами.
     *
     * @param v DTO тербанка с реквизитами
     * @return код тербанка
     */
    @Override
    public String extractKey(TerBankWithRequisiteDto v) {
        return v.getTbCode();
    }

    /**
     * Загружает тербанки с реквизитами по списку кодов.
     *
     * @param codes список кодов тербанков
     * @return загруженные DTO тербанков с реквизитами
     */
    @Override
    public List<TerBankWithRequisiteDto> fetchByKeys(List<String> codes) {
        var resp = master.requestDataWithAttribute(
            props.getSlugValueForTerBank(),
            codes,
            SearchRequestProperties.Context.BOOK
        );
        return createWithAttribute(resp, mapper);
    }
}

```
