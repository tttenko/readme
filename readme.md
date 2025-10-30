```java

Метод requestDataWithRefItemSlug
/**
 * Выполняет запрос к мастер-данным по ссылочному атрибуту (refItemSlug).
 * <p>Используется для получения связанных сущностей по списку идентификаторов.</p>
 *
 * @param dictionaryName имя справочника (slug)
 * @param refItemSlug идентификатор ссылочного атрибута
 * @param ids список идентификаторов элементов для выборки
 * @return ответ от мастер-данных в виде {@link GetItemsSearchResponse}
 * @see RequestFactory#getByAttrValuesBuilder()
 * @since 1.0
 */

Метод requestDataWithAttribute
/**
 * Выполняет запрос к мастер-данным с фильтрацией по атрибутам.
 * <p>Возвращает элементы справочника с указанными атрибутами и их значениями.</p>
 *
 * @param dictionaryName имя справочника (slug)
 * @param filters карта фильтров (атрибут → список значений)
 * @return ответ от мастер-данных в виде {@link GetItemsSearchResponse}
 * @see RequestFactory#getByAttrValuesBuilder()
 * @since 1.0
 */
```
