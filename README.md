```java

@Schema(
    description = "Статус привязки метрики к инициативе",
    type = "string",
    nullable = false,
)
val applicabilityStatus: MetricApplicabilityStatus = MetricApplicabilityStatus.ACTIVE,

@Schema(
    description = "Период окончания отвязки метрики от инициативы",
    type = "string",
    format = "date",
    nullable = true,
)
val resumePeriod: LocalDate? = null,

@Schema(
    description = "Идентификатор актуальной заявки на неприменимость метрики",
    type = "number",
    format = "int64",
    nullable = true,
)
val applicabilityRequestId: Long? = null,



```
