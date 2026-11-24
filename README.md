```java
1. MetaAiObjectsResponse.kt
import io.swagger.v3.oas.annotations.media.Schema

@Schema(description = "Список связанных AI-агентов и LLM-решений инициативы")
data class MetaAiObjectsResponse(
    @field:Schema(description = "Активные связанные AI-объекты инициативы. Если объекты отсутствуют, возвращается пустой массив")
    val objects: List<MetaAiObjectResponse>
)


2. MetaAiObjectResponse.kt
import io.swagger.v3.oas.annotations.media.Schema

@Schema(description = "Данные связанного AI-агента или LLM-решения")
data class MetaAiObjectResponse(
    @field:Schema(description = "ID записи связанного AI-объекта", example = "512")
    val id: Long,

    @field:Schema(description = "КЭ связанного объекта", example = "CI12345678", nullable = true)
    val ci: String?,

    @field:Schema(description = "Наименование AI-объекта", example = "Ассистент кассовых операций", nullable = true)
    val name: String?,

    @field:Schema(description = "Тип объекта: AGENT — ИИ-агент, LLM_SOLUTION — LLM-решение, null — тип не определён", example = "AGENT", nullable = true)
    val type: MetaAiObjectType?,

    @field:Schema(description = "Идентификатор объекта в GigaUsage", example = "GIGAUSAGE-123456", nullable = true)
    val gigaUsageId: String?,

    @field:Schema(description = "Прямая ссылка на объект в META", example = "https://meta.sber.ru/objects/512", nullable = true)
    val metaLink: String?,

    @field:Schema(description = "Количество активных отклонений связанного объекта", example = "2")
    val deviationCount: Int,

    @field:Schema(description = "Список активных отклонений объекта, отсортированных по приоритету")
    val deviations: List<MetaAiObjectDeviationResponse>
)


3. MetaAiObjectDeviationResponse.kt
import io.swagger.v3.oas.annotations.media.Schema

@Schema(description = "Отклонение связанного AI-объекта")
data class MetaAiObjectDeviationResponse(
    @field:Schema(description = "Уникальный технический код отклонения", example = "AI_OBJECT_NAME_NOT_FILLED")
    val code: String,

    @field:Schema(description = "Заголовок отклонения из backend-конфигурации", example = "Не заполнено наименование")
    val title: String,

    @field:Schema(description = "Описание отклонения из backend-конфигурации", example = "Для агента или LLM-решения не указано наименование")
    val description: String,

    @field:Schema(description = "Приоритет сортировки отклонений. Чем меньше значение, тем выше отклонение в списке", example = "20")
    val priority: Int
)


4. MetaAiObjectType.kt
Для enum тоже добавим описание, чтобы в Swagger было понятно назначение каждого значения.
import io.swagger.v3.oas.annotations.media.Schema

@Schema(description = "Тип связанного AI-объекта: AGENT — ИИ-агент, LLM_SOLUTION — LLM-решение")
enum class MetaAiObjectType {
    AGENT,
    LLM_SOLUTION
}
```
