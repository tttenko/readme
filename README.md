```java

interface MetricDataForExportView {

    fun getAgentId(): Long?

    fun getBlock(): String?

    fun getDivision(): String?

    fun getName(): String?

    fun getCrossgoal(): String?

    fun getType(): String?

    fun getMetric(): String?

    fun getPeriodType(): String?

    fun getApplicabilityStatus(): String?

    fun getResumePeriod(): LocalDate?

    fun getPeriodMonth(): LocalDate?

    fun getTargetValue(): BigDecimal?

    fun getMetricValue(): BigDecimal?
}

@Query(
        value = """
            select 
                a.id as agent_id,
                b.short_name as block,
                d.short_name as division,
                a.agent_name as name,
                ji.jira_key as crossgoal,
                mt.agent_type as type,
                md.name as metric,
                md.frequency as periodType,

                assignment.applicability_status as applicabilityStatus,
                actual_request.resume_period as resumePeriod,

                mv.period_month as period_month,
                mv.target_value as target_value,
                mv.metric_value as metric_value

            from initiative_metric_value mv

                left join initiative_metric_type mt
                    on mt.id = mv.initiative_agent_type_id

                left join metrics_directory md
                    on md.id = mv.metric_directory_id

                left join ai_agent a
                    on a.id = mt.ai_agent_id

                left join block b
                    on b.id = a.block_id

                left join division d
                    on d.id = a.division_id

                left join jira_issue ji
                    on a.id = ji.agent_id
                    and ji.type = 'initiative'
                    and ji.project = 'crossgoal'

                left join initiative_metric_assignment assignment
                    on assignment.initiative_agent_type_id = mt.id
                    and assignment.metric_directory_id = md.id

                left join lateral (
                    select request.resume_period
                    from metric_applicability_request request
                    where request.initiative_metric_assignment_id = assignment.id
                    order by request.created_at desc, request.id desc
                    limit 1
                ) actual_request on true

            where (:blockId is null or a.block_id = :blockId)
              and (:divisionId is null or a.division_id = :divisionId)
              and mv.period_month between :periodFrom and :periodTo
        """,
        nativeQuery = true
    )
    fun getMetricDataForExport(
        @Param("periodFrom")
        periodFrom: LocalDate,

        @Param("periodTo")
        periodTo: LocalDate,

        @Param("divisionId")
        divisionId: Long?,

        @Param("blockId")
        blockId: Long?
    ): List<MetricDataForExportView>

data class MetricsExcelExportModel(
    val block: String?,
    val division: String?,
    val name: String?,
    val crossgoal: String?,
    val type: String?,
    val metric: String?,
    val periodType: String?,

    val notApplicable: String?,

    val data: Map<YearMonth?, BigDecimal?>,
    val planData: Map<YearMonth?, BigDecimal?>
)

@Service
class MetricsReportService(
    private val metricsRepository: MetricRepository
) {

    companion object {

        private val PERIOD_FORMATTER =
            DateTimeFormatter.ofPattern("MM-yyyy")

        private val DATE_FORMATTER =
            DateTimeFormatter.ofPattern("dd.MM.yyyy")
    }

    fun downloadExcelReport(
        periodFrom: LocalDate,
        periodTo: LocalDate,
        divisionId: Long?,
        blockId: Long?
    ): ResponseEntity<InputStreamResource> {

        // Подготовка таблицы Excel.
        val workBook =
            ExcelExportHelper.createWorkBook(
                listOf("Метрики")
            )

        // Получение и подготовка данных для выгрузки.
        val data =
            toMetricsExcelExportModel(
                metricsRepository.getMetricDataForExport(
                    periodFrom = periodFrom,
                    periodTo = periodTo,
                    divisionId = divisionId,
                    blockId = blockId
                )
            ).ifEmpty {
                return ResponseEntity.noContent().build()
            }

        // Заполнение данных.
        ExcelExportHelper.writeSheetData(
            workBook = workBook,
            sheet = workBook.getSheetAt(0),
            data = data,
            columnDescriptions =
                headerColumns(
                    getYearMonthsBetween(
                        periodFrom,
                        periodTo
                    )
                )
        )

        // Превращение таблицы в файл.
        return workBook.convertToFile(
            "metrics_export_${periodFrom}_${periodTo}.xlsx"
        )
    }

    /**
     * Преобразует строки projection,
     * полученные из БД, в строки Excel.
     *
     * Несколько записей одной метрики отличаются периодом,
     * поэтому сначала группируем их по:
     *
     * initiative + agentType + metric.
     */
    fun toMetricsExcelExportModel(
        data: List<MetricDataForExportView>
    ): List<MetricsExcelExportModel> {

        return data
            .groupBy { metricData ->
                Triple(
                    metricData.getAgentId(),
                    metricData.getType(),
                    metricData.getMetric()
                )
            }
            .mapNotNull { (_, metricValues) ->

                val first =
                    metricValues.firstOrNull()
                        ?: return@mapNotNull null

                MetricsExcelExportModel(
                    block = first.getBlock(),

                    division = first.getDivision(),

                    name = first.getName(),

                    crossgoal = first.getCrossgoal(),

                    type =
                        first.getType()?.let { agentType ->
                            when (
                                InitiativeMetricAgentType.fromValue(agentType)
                            ) {
                                COPILOT ->
                                    COPILOT.value

                                APPEALS ->
                                    "работа с обращениями"

                                else ->
                                    "автономный"
                            }
                        },

                    metric = first.getMetric(),

                    periodType = first.getPeriodType(),

                    /*
                     * Новая колонка "Неприменима".
                     */
                    notApplicable =
                        getNotApplicableValue(
                            applicabilityStatus =
                                first.getApplicabilityStatus(),
                            resumePeriod =
                                first.getResumePeriod()
                        ),

                    data =
                        metricValues.associate { metricData ->
                            metricData
                                .getPeriodMonth()
                                ?.let(YearMonth::from) to
                                    metricData.getMetricValue()
                        },

                    planData =
                        metricValues.associate { metricData ->
                            metricData
                                .getPeriodMonth()
                                ?.let(YearMonth::from) to
                                    metricData.getTargetValue()
                        }
                )
            }
    }

    /**
     * Формирует отображаемое значение
     * колонки "Неприменима".
     *
     * assignment отсутствует -> пусто
     * ACTIVE -> пусто
     * PENDING -> "Согласование"
     * NOT_APPLICABLE без resumePeriod -> "Да"
     * NOT_APPLICABLE с resumePeriod ->
     * "До <последний день месяца>"
     */
    private fun getNotApplicableValue(
        applicabilityStatus: String?,
        resumePeriod: LocalDate?
    ): String? {

        return when (applicabilityStatus) {

            null,
            MetricApplicabilityStatus.ACTIVE.name ->
                null

            MetricApplicabilityStatus.PENDING.name ->
                "Согласование"

            MetricApplicabilityStatus.NOT_APPLICABLE.name -> {

                if (resumePeriod == null) {
                    "Да"
                } else {

                    val lastDayOfResumeMonth =
                        YearMonth
                            .from(resumePeriod)
                            .atEndOfMonth()

                    "До ${
                        lastDayOfResumeMonth.format(
                            DATE_FORMATTER
                        )
                    }"
                }
            }

            else ->
                null
        }
    }

    /**
     * Описание колонок Excel.
     */
    private fun headerColumns(
        periods: Set<YearMonth>
    ): List<ExcelColumnDescription<MetricsExcelExportModel>> {

        return mutableListOf(

            ExcelColumnDescription(
                "Блок",
                { param ->
                    param.first.block?.let {
                        param.third.setCellValue(it)
                    }
                }
            ),

            ExcelColumnDescription(
                "Трайб",
                { param ->
                    param.first.division?.let {
                        param.third.setCellValue(it)
                    }
                }
            ),

            ExcelColumnDescription(
                "Название агента",
                { param ->
                    param.first.name?.let {
                        param.third.setCellValue(it)
                    }
                }
            ),

            ExcelColumnDescription(
                "CROSSGOAL",
                { param ->
                    param.first.crossgoal?.let {
                        param.third.setCellValue(it)
                    }
                }
            ),

            ExcelColumnDescription(
                "Тип агента (автономный/copilot)",
                { param ->
                    param.first.type?.let {
                        param.third.setCellValue(it)
                    }
                }
            ),

            ExcelColumnDescription(
                "Метрика",
                { param ->
                    param.first.metric?.let {
                        param.third.setCellValue(it)
                    }
                }
            ),

            ExcelColumnDescription(
                "Периодичность сбора " +
                    "(регулярный мониторинг/вводный параметр)",
                { param ->
                    param.first.periodType?.let {
                        param.third.setCellValue(it)
                    }
                }
            ),

            /*
             * Новая колонка.
             */
            ExcelColumnDescription(
                "Неприменима",
                { param ->
                    param.first.notApplicable?.let {
                        param.third.setCellValue(it)
                    }
                }
            )

        ).also { columns ->

            periods.forEach { period ->

                columns.add(
                    ExcelColumnDescription(
                        "факт / ${period.format(PERIOD_FORMATTER)}",
                        { param ->
                            param.first.data[period]?.let {
                                param.third.setCellValue(
                                    it.toDouble()
                                )
                            }
                        }
                    )
                )

                columns.add(
                    ExcelColumnDescription(
                        "план / ${period.format(PERIOD_FORMATTER)}",
                        { param ->
                            param.first.planData[period]?.let {
                                param.third.setCellValue(
                                    it.toDouble()
                                )
                            }
                        }
                    )
                )
            }
        }
    }
}

```
