```java
/**
 * Реестр маршрутов обработки сообщений.
 * <p>
 * Собирает маршруты из провайдеров ({@link KafkaRoutesProvider}) и предоставляет доступ к ним по ключу.
 */
public class RoutesRegistry {

    /**
     * Создает реестр маршрутов и индексирует их по ключу.
     *
     * @param providers список провайдеров маршрутов
     * @throws IllegalStateException если обнаружены дублирующиеся маршруты с одинаковым ключом
     */
    public RoutesRegistry(List<? extends KafkaRoutesProvider> providers) {
    }

    /**
     * Возвращает маршрут по ключу.
     *
     * @param key ключ маршрута
     * @param <T> тип DTO, который ожидает consumer маршрута
     * @return определение маршрута
     * @throws UnsupportedOperationException если маршрут по ключу не найден
     */
    public <T> RouteDefinition<T> getRequired(String key) {
        return null;
    }
}
import java.util.List;

/**
 * Провайдер набора маршрутов для обработки сообщений.
 * <p>
 * Используется для декларативного описания доступных маршрутов и их обработчиков.
 */
public interface KafkaRoutesProvider {

    /**
     * Возвращает список определений маршрутов.
     *
     * @return список маршрутов
     */
    List<RouteDefinition<?>> routes();
}
/**
 * Маркерный интерфейс провайдера маршрутов для домена курсов/ставок.
 * <p>
 * Нужен для группировки и автоматического подхвата соответствующих провайдеров в реестр.
 */
public interface RateKafkaRoutesProvider extends KafkaRoutesProvider {
}
import java.util.function.Consumer;

/**
 * Определение маршрута обработки сообщения.
 *
 * @param <T> тип DTO, который будет передан в обработчик
 */
public class RouteDefinition<T> {

    /**
     * Создает определение маршрута.
     *
     * @param key      ключ маршрута (идентификатор)
     * @param dtoClass класс DTO для десериализации/валидации
     * @param consumer обработчик сообщения
     */
    public RouteDefinition(String key, Class<T> dtoClass, Consumer<T> consumer) {
    }

    /**
     * Фабричный метод для удобного создания маршрута.
     *
     * @param key      ключ маршрута
     * @param dtoClass класс DTO
     * @param consumer обработчик
     * @param <T>      тип DTO
     * @return определение маршрута
     */
    public static <T> RouteDefinition<T> route(String key, Class<T> dtoClass, Consumer<T> consumer) {
        return null;
    }

    /**
     * Возвращает ключ маршрута.
     *
     * @return ключ
     */
    public String getKey() {
        return null;
    }

    /**
     * Возвращает класс DTO маршрута.
     *
     * @return класс DTO
     */
    public Class<T> getDtoClass() {
        return null;
    }

    /**
     * Возвращает обработчик сообщения.
     *
     * @return consumer
     */
    public Consumer<T> getConsumer() {
        return null;
    }
}
import java.util.List;

/**
 * Реестр маршрутов обработки для домена курсов валют/металлов.
 * <p>
 * Специализация {@link RoutesRegistry}, которая собирает маршруты из {@link RateKafkaRoutesProvider}.
 */
public class CurrencyRateRoutesRegistry extends RoutesRegistry {

    /**
     * Создает реестр маршрутов курсов и регистрирует их из провайдеров.
     *
     * @param providers список провайдеров маршрутов курсов
     */
    public CurrencyRateRoutesRegistry(List<RateKafkaRoutesProvider> providers) {
        super(providers);
    }
}

```
