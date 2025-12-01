```java

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Schema(description = "Статистика по одному кэшу")
public class CacheStatusDto {

    @Schema(description = "Имя кэша")
    private String name;

    @Schema(description = "Тип кэша (класс Spring Cache)")
    private String type;

    @Schema(description = "Текущий размер (estimatedSize)")
    private Long estimatedSize;

    @Schema(description = "Кол-во попаданий (hitCount)")
    private Long hitCount;

    @Schema(description = "Кол-во промахов (missCount)")
    private Long missCount;

    @Schema(description = "Доля попаданий (hitRate)")
    private Double hitRate;

    @Schema(description = "Доля промахов (missRate)")
    private Double missRate;

    @Schema(description = "Успешные загрузки (loadSuccessCount)")
    private Long loadSuccess;

    @Schema(description = "Неуспешные загрузки (loadFailureCount)")
    private Long loadFailure;

    @Schema(description = "Удаления по политике кэша (evictionCount)")
    private Long evictionCount;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Schema(description = "Общий статус кэшей приложения")
public class CacheStatusResponse {

    @Schema(description = "Последняя ручная инвалидация всех кэшей")
    private String lastManualInvalidation;

    @Schema(description = "TTL кэшей (в минутах)")
    private long ttl;

    @Schema(description = "Статистика по каждому кэшу")
    private List<CacheStatusDto> caches;
}

```
