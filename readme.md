```java

package ru.sber.cs.supplier.portal.masterdata.support;

import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.ArgumentMatchers.isNull;
import static org.mockito.Mockito.when;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import org.mockito.stubbing.OngoingStubbing;
import ru.sber.cs.supplier.portal.masterdata.api.model.GetItemsSearchResponse;
import ru.sber.cs.supplier.portal.masterdata.api.model.ResponseMessage;
import ru.sber.cs.supplier.portal.masterdata.services.impl.BaseMasterDataRequestService;
import ru.sber.cs.supplier.portal.masterdata.services.props.SearchRequestProperties;

public final class MasterDataFixtures {

  // JSON-ключи, которые ожидает маппинг в сервисе
  private static final String KEY_ITEMS = "items";
  private static final String KEY_ITEM = "item";
  private static final String KEY_VALUES = "values";
  private static final String KEY_ATTRIBUTE = "attribute";
  private static final String KEY_SLUG = "slug";
  private static final String KEY_VALUE = "value";
  private static final String KEY_NAME = "name";

  private MasterDataFixtures() {}

  /** Сообщение "успех" для checkResponseStatus(...). По умолчанию semantic="S". */
  public static ResponseMessage okMessage() {
    ResponseMessage message = new ResponseMessage();
    message.setSemantic("S"); // если у вас SUCCESS ≠ "S", подставьте реальное значение
    message.setMessage("OK");
    message.setDescription("OK");
    return message;
  }

  /** Успешный ответ БЕЗ атрибутов: data -> [{ items: [ { slug, name } ] }]. */
  public static GetItemsSearchResponse okNoAttrResponse(final String code, final String name) {
    GetItemsSearchResponse response = new GetItemsSearchResponse();
    response.setMessages(List.of(okMessage()));

    Map<String, Object> item = new HashMap<>();
    item.put(KEY_SLUG, code);
    item.put(KEY_NAME, name);

    Map<String, Object> page = new HashMap<>();
    page.put(KEY_ITEMS, List.of(item));

    response.setData(List.of(page));
    return response;
  }

  /**
   * Успешный ответ С атрибутами:
   * data -> [{ items: [ { item:{slug,name}, values:[ {attribute:{slug:<attr>}, value:<val>} ... ] } ] }]
   */
  public static GetItemsSearchResponse okWithAttrResponse(final String code,
                                                          final String name,
                                                          final Map<String, String> attributes) {
    GetItemsSearchResponse response = new GetItemsSearchResponse();
    response.setMessages(List.of(okMessage()));

    Map<String, Object> itemValues = new HashMap<>();
    itemValues.put(KEY_SLUG, code);
    itemValues.put(KEY_NAME, name);

    // values[]
    List<Map<String, Object>> valuesArray =
        attributes.entrySet().stream()
            .map(e -> {
              Map<String, Object> attrNode = new HashMap<>();
              attrNode.put(KEY_ATTRIBUTE, Map.of(KEY_SLUG, e.getKey()));
              attrNode.put(KEY_VALUE, e.getValue());
              return attrNode;
            })
            .toList();

    Map<String, Object> itemNode = new HashMap<>();
    itemNode.put(KEY_ITEM, itemValues);
    itemNode.put(KEY_VALUES, valuesArray);

    Map<String, Object> page = new HashMap<>();
    page.put(KEY_ITEMS, List.of(itemNode));

    response.setData(List.of(page));
    return response;
  }

  // ---------- Удобные стабы Mockito ----------

  public static OngoingStubbing<GetItemsSearchResponse> stubRequestDataOk(
      final BaseMasterDataRequestService service,
      final String dictionarySlug,
      final SearchRequestProperties.Context context,
      final GetItemsSearchResponse response) {

    return when(service.requestData(eq(dictionarySlug), isNull(), eq(context))).thenReturn(response);
  }

  public static OngoingStubbing<GetItemsSearchResponse> stubRequestDataWithAttrOk(
      final BaseMasterDataRequestService service,
      final String dictionarySlug,
      final SearchRequestProperties.Context context,
      final GetItemsSearchResponse response) {

    return when(service.requestDataWithAttribute(eq(dictionarySlug), isNull(), eq(context)))
        .thenReturn(response);
  }
}

```
