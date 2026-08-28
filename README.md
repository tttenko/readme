```java

data class InitiativeMetricPeriodResponse(
    @JsonFormat(
        shape = JsonFormat.Shape.STRING,
        pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'",
        timezone = "UTC",
    )
    val period: Instant,

    val value: BigDecimal?,
)
```
