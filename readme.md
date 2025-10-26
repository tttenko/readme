```java

/**
 * Набор тестовых фикстур для проверки кеширования и интеграции со слоем запросов master-данных.
 * <p>
 * Предоставляет:
 * <ul>
 *   <li>Генерацию успешного {@code ResponseMessage} для {@code checkResponseStatus(...)}.</li>
 *   <li>Конструирование детерминированных ответов {@code GetItemsSearchResponse}
 *       без атрибутов и с атрибутами в формате, ожидаемом мапперами (ключи: {@code items},
 *       {@code item}, {@code values}, {@code attribute}, {@code slug}, {@code value}, {@code name}).</li>
 *   <li>Удобные стабы для {@link ru.sber.cs.supplier.portal.masterdata.services.impl.BaseMasterDataRequestService}
 *       c Mockito: {@code requestData(...)} и {@code requestDataWithAttribute(...)}.</li>
 * </ul>
 * Используется в unit/integ-тестах кеша для имитации корректных ответов внешнего сервиса.
 */

 /**
 * Возвращает сообщение «успех» для проверки статуса ответа внешнего сервиса.
 * <p>По умолчанию semantic = {@code "S"}, text = {@code "OK"}, description = {@code "OK"}.</p>
 *
 * @return заполненный {@link ResponseMessage} с признаками успешного ответа
 */

 /**
 * Формирует успешный ответ БЕЗ атрибутов для сценариев {@code requestData(...)}.
 * <p>Структура {@code data}: {@code [{ "items": [ { "slug": code, "name": name } ] }]}.</p>
 *
 * @param code код/slug сущности (будет помещён в поле {@code slug})
 * @param name отображаемое имя сущности (будет помещено в поле {@code name})
 * @return объект {@link GetItemsSearchResponse} с сообщением «успех» и одним элементом без атрибутов
 */

 /**
 * Формирует успешный ответ С атрибутами для сценариев {@code requestDataWithAttribute(...)}.
 * <p>Структура {@code data}:
 * <pre>
 * [
 *   {
 *     "items": [
 *       {
 *         "item":   { "slug": code, "name": name },
 *         "values": [
 *           { "attribute": { "slug": attrSlug }, "value": attrValue },
 *           ...
 *         ]
 *       }
 *     ]
 *   }
 * ]
 * </pre>
 * где пары {@code attributes} преобразуются в элементы массива {@code values}.
 * </p>
 *
 * @param code        код/slug сущности
 * @param name        отображаемое имя сущности
 * @param attributes  карта атрибутов: ключ — {@code attribute.slug}, значение — {@code value}
 * @return объект {@link GetItemsSearchResponse} с сообщением «успех» и одним элементом с атрибутами
 */

/**
 * Ставит стаб на {@code BaseMasterDataRequestService.requestData(...)} с успешным ответом БЕЗ атрибутов.
 * <p>Ожидаемые аргументы: {@code dictionarySlug}, {@code null} как searchFieldValue (семантика «все»),
 * {@code context}.</p>
 *
 * @param service         мок сервиса master-данных
 * @param dictionarySlug  ожидаемый slug справочника (например, {@code "terbank-slug"})
 * @param context         ожидаемый контекст поиска
 * @param response        ответ, который должен вернуть мок
 * @return {@link org.mockito.stubbing.OngoingStubbing} для дальнейшей донастройки при необходимости
 */

 /**
 * Ставит стаб на {@code BaseMasterDataRequestService.requestDataWithAttribute(...)} с успешным ответом С атрибутами.
 * <p>Ожидаемые аргументы: {@code dictionarySlug}, {@code null} как searchFieldValue (семантика «все»),
 * {@code context}.</p>
 *
 * @param service         мок сервиса master-данных
 * @param dictionarySlug  ожидаемый slug справочника
 * @param context         ожидаемый контекст поиска
 * @param response        ответ, который должен вернуть мок (с атрибутами)
 * @return {@link org.mockito.stubbing.OngoingStubbing} для дальнейшей донастройки при необходимости
 */


```
