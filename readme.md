```java
/**
 * Исключение, сигнализирующее о некорректном или непригодном для обработки XML
 * в сообщениях с курсами валют (например, не удалось определить корневой тег/маршрут).
 * <p>
 * Используется для единообразной обработки ошибок парсинга/валидации входного XML.
 */
public class InvalidCurrencyRateXmlException extends RuntimeException {

    /**
     * Создаёт исключение с сообщением и первопричиной.
     *
     * @param message описание ошибки
     * @param cause исходное исключение (первопричина)
     */
    public InvalidCurrencyRateXmlException(String message, Throwable cause) { /* ... */ }

    /**
     * Создаёт исключение только с сообщением.
     *
     * @param message описание ошибки
     */
    public InvalidCurrencyRateXmlException(String message) { /* ... */ }
}

/**
 * DTO с информацией о курсе валюты/драгоценного металла.
 * <p>
 * Содержит исходную и целевую валюту (коды и ISO-номера), дату начала действия курса,
 * значение курса и коэффициент лота, а также рассчитанный курс за 1 единицу.
 */

 /**
 * Возвращает курс валюты/металла по параметрам {@code isoNum1}, {@code isoNum2} и {@code date}.
 * <p>
 * Принимает дату в формате {@code dd.MM.yyyy} и числовые ISO-коды исходной и целевой валют.
 *
 * @param date дата, на которую требуется курс (формат {@code dd.MM.yyyy})
 * @param isoNum1 числовой ISO-код исходной валюты
 * @param isoNum2 числовой ISO-код целевой валюты
 * @return обёртка {@link ResultObj} со списком {@link CurrencyRateDto} (обычно 0 или 1 элемент)
 */
@GetMapping(value = "/currencyRate", produces = APPLICATION_JSON_VALUE)
@Operation(operationId = "getCurrencyRate", summary = "Получение курса по параметрам (isoNum1, isoNum2, date)")
ResultObj<List<CurrencyRateDto>> getCurrencyRate(
        /* ... */
);

/**
 * Обрабатывает HTTP-запрос на получение курса по параметрам {@code date}, {@code isoNum1} и {@code isoNum2}.
 * <p>
 * Делегирует поиск в {@code currencyService.getCurrencyRate(...)} и формирует {@link ResultObj}:
 * заполняет данные найденными курсами, количество записей и сообщение об успешном выполнении
 * с описанием результата поиска.
 *
 * @param date дата, на которую требуется курс
 * @param isoNum1 числовой ISO-код исходной валюты
 * @param isoNum2 числовой ISO-код целевой валюты
 * @return {@link ResultObj} со списком {@link CurrencyRateDto} и метаданными результата
 */
@Override
public ResultObj<List<CurrencyRateDto>> getCurrencyRate(
        @RequestParam("date") LocalDate date,
        @RequestParam("isoNum1") String isoNum1,
        @RequestParam("isoNum2") String isoNum2
) { /* ... */ }
```
