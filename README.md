```java

@Getter
@Setter
@Schema(description = "Данные СТС")
public class StsDataDto {

    @Schema(description = "Уникальный идентификатор записи")
    private UUID uuid;

    @Schema(description = "Uuid договора")
    private UUID contractUuid;

    @Schema(description = "Код ТБ")
    private String tbId;

    @Schema(description = "Номер ТС")
    private String vehicleNumber;

    @Schema(description = "Марка ТС")
    private String vehicleBrand;

    @Schema(description = "Примечание")
    private String comment;

    @Schema(description = "Код статуса")
    private String statusId;

    @Schema(description = "Код автора создания")
    private String createdBy;

    @Schema(description = "Код автора изменения")
    private String changedBy;

    @Schema(description = "Дата создания")
    private LocalDateTime createdAt;

    @Schema(description = "Дата последнего изменения")
    private LocalDateTime changedAt;

    @Schema(description = "Признак удаления")
    private boolean deleted;
}



```
