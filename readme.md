```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@JsonIgnoreProperties(ignoreUnknown = true)
@Schema(description = "Информация о дате из производственного календаря")
public class ProdCalendDateDto implements Serializable {

    @Schema(description = "UUID записи")
    private String id;

    @Schema(description = "Дата из МД (например: YYYY-MM-DD 00:00:00.0)")
    private String date;

    @Schema(description = "Короткая дата (dd.MM.yyyy)")
    private String dateShort;

    @Schema(description = "Код типа дня (например: 1 - рабочий, 4 - нерабочий)")
    private String dateType;

    @Schema(description = "Описание типа дня")
    private String dayTypeDescription;
}
```
