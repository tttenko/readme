```java

@Getter
@Setter
@Schema(description = "Страница с данными")
public class PageDto<T> {

    @Schema(description = "Элементы страницы")
    private List<T> items;

    @Schema(description = "Номер страницы")
    private int page;

    @Schema(description = "Размер страницы")
    private int size;

    @Schema(description = "Общее количество элементов")
    private long totalElements;

    @Schema(description = "Общее количество страниц")
    private int totalPages;

    @Schema(description = "Признак первой страницы")
    private boolean first;

    @Schema(description = "Признак последней страницы")
    private boolean last;
}

    public static <T> ResultObj<PageDto<T>> getSuccessPageResponse(PageDto<T> page) {
        ResultObj<PageDto<T>> result = new ResultObj<>();
        result.addMessages(success("Поиск выполнен", null, "Найдено записей: " + page.getTotalElements()));
        result.setData(page);
        result.setCount(page.getTotalElements());
        return result;
    }
```
