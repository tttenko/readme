```java
/**
 * Определение маршрута обработки сообщения.
 *
 * @param <T> тип DTO, который будет передан в обработчик
 */
@Getter
@AllArgsConstructor
public class RouteDefinition<T> {

    /**
     * Ключ маршрута (идентификатор), по которому выбирается обработчик.
     */
    private final String key;

    /**
     * Класс DTO для десериализации/валидации входных данных маршрута.
     */
    private final Class<T> dtoClass;

    /**
     * Обработчик (consumer), который будет вызван для DTO указанного типа.
     */
    private final Consumer<T> consumer;

    /**
     * Фабричный метод для создания {@link RouteDefinition}.
     *
     * @param key ключ маршрута (идентификатор)
     * @param dtoClass класс DTO маршрута
     * @param consumer обработчик DTO
     * @param <T> тип DTO
     * @return новый экземпляр {@link RouteDefinition}
     */
    public static <T> RouteDefinition<T> route(String key, Class<T> dtoClass, Consumer<T> consumer) { /* ... */ }

}

```
