```java

enum class InitiativeDeviationCode {
    STAGE_DEADLINE_EXPIRED,
    CURRENT_PERIOD_METRICS_NOT_FILLED,
    STAGE_DEADLINES_NOT_FILLED,
    GIGAUSAGE_NOT_FILLED,
    ENABLERS_NOT_FILLED,
    STAGE_DEADLINE_TOMORROW,
    STAGE_DEADLINE_IN_2_DAYS,
    STAGE_DEADLINE_IN_3_DAYS,
}

@ConfigurationProperties(prefix = "initiative-deviations")
data class InitiativeDeviationProperties(

    /**
     * После какого дня месяца начинает проверяться заполненность метрик.
     *
     * При значении 15 проверка выполняется начиная с 16-го числа.
     */
    val currentPeriodMetricsDeadlineDay: Int = 15,

    /**
     * Настройки правил по коду отклонения.
     */
    val rules: Map<InitiativeDeviationCode, Rule> = emptyMap(),
) {

    fun getRequiredRule(code: InitiativeDeviationCode): Rule =
        requireNotNull(rules[code]) {
            "Не найдена конфигурация отклонения $code"
        }

    data class Rule(
        val enabled: Boolean = true,
        val priority: Int,
        val title: String,
        val description: String,
        val cta: String? = null,
    )
}

initiative-deviations:
  current-period-metrics-deadline-day: 15

  rules:
    STAGE_DEADLINE_EXPIRED:
      enabled: true
      priority: 1
      title: "Нарушен дедлайн этапа"
      description: "Срок завершения одного из текущих этапов уже прошёл"
      cta: "Изменить сроки"

    CURRENT_PERIOD_METRICS_NOT_FILLED:
      enabled: true
      priority: 2
      title: "Внесите метрики за текущий период"
      description: "Заполнены не все обязательные метрики за текущий месяц"
      cta: "Внести"

    STAGE_DEADLINES_NOT_FILLED:
      enabled: true
      priority: 3
      title: "Укажите дедлайны этапов"
      description: "Для одного или нескольких незавершённых этапов не указан срок"
      cta: "Указать"

    GIGAUSAGE_NOT_FILLED:
      enabled: true
      priority: 4
      title: "Добавьте GigaUsage"
      description: "Для инициативы не создана задача GigaUsage"
      cta: "Добавить"

    ENABLERS_NOT_FILLED:
      enabled: true
      priority: 5
      title: "Добавьте энейблеры"
      description: "Для инициативы не указан ни один энейблер"
      cta: "Добавить"

    STAGE_DEADLINE_TOMORROW:
      enabled: true
      priority: 6
      title: "Завтра дедлайн этапа"
      description: "Срок завершения одного из этапов наступает завтра"
      cta: "Проверить статус"

    STAGE_DEADLINE_IN_2_DAYS:
      enabled: true
      priority: 7
      title: "Через 2 дня дедлайн этапа"
      description: "Срок завершения одного из этапов наступает через два дня"
      cta: "Проверить статус"

    STAGE_DEADLINE_IN_3_DAYS:
      enabled: true
      priority: 8
      title: "Через 3 дня дедлайн этапа"
      description: "Срок завершения одного из этапов наступает через три дня"
      cta: "Проверить статус"

data class InitiativeDeviationResponse(

    @field:Schema(
        description = "Код отклонения",
        example = "STAGE_DEADLINE_EXPIRED",
    )
    val code: String,

    @field:Schema(
        description = "Приоритет отображения",
        example = "1",
    )
    val priority: Int,

    @field:Schema(
        description = "Заголовок отклонения",
        example = "Нарушен дедлайн этапа",
    )
    val title: String,

    @field:Schema(
        description = "Описание отклонения",
    )
    val description: String,
)

data class InitiativeDeviationEvaluationContext(
    val today: LocalDate,
    val currentPeriod: LocalDate,
    val metricsDeadlineDay: Int,
)

@Configuration
class TimeConfiguration {

    @Bean
    fun clock(): Clock = Clock.systemUTC()
}



```
