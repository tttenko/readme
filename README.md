```java

@Service
class MetricsReportService(
    private val metricsRepository: MetricRepository
) {

    companion object {
        private val PERIOD_FORMATTER = DateTimeFormatter.ofPattern("MM-yyyy")
        private val DATE_FORMATTER = DateTimeFormatter.ofPattern("dd.MM.yyyy")
    }

    fun downloadExcelReport(
        periodFrom: LocalDate,
        periodTo: LocalDate,
        divisionId: Long?,
        blockId: Long?
    ): ResponseEntity<InputStreamResource> {

        // Подготовка таблицы excel
        val workBook = ExcelExportHelper.createWorkBook(listOf("Метрики"))

        // Получение и подготовка данных для выгрузки
        val data = toMetricsExcelExportModel(
            metricsRepository.getMetricDataForExport(periodFrom, periodTo, divisionId, blockId)
        ).ifEmpty { return ResponseEntity.noContent().build() }

        // Заполнение данных
        ExcelExportHelper.writeSheetData(
            workBook,
            workBook.getSheetAt(0),
            data,
            headerColumns(getYearMonthsBetween(periodFrom, periodTo))
        )

        // Превращение полученной таблицы в файл
        return workBook.convertToFile("metrics_export_${periodFrom}_${periodTo}.xlsx")
    }

    fun toMetricsExcelExportModel(data: List<MetricDataForExportView>): List<MetricsExcelExportModel> {
        return data
            .groupBy {
                Triple(
                    it.getAgentId(),
                    it.getType(),
                    it.getMetric()
                )
            }
            .mapNotNull { agentMetrics ->
                agentMetrics.value.firstOrNull()?.let { metricData ->
                    MetricsExcelExportModel(
                        block = metricData.getBlock(),
                        division = metricData.getDivision(),
                        name = metricData.getName(),
                        crossgoal = metricData.getCrossgoal(),
                        type = metricData.getType()?.let { agentType ->
                            when (InitiativeMetricAgentType.fromValue(agentType)) {
                                COPILOT -> COPILOT.value
                                APPEALS -> "работа с обращениями"
                                else -> "автономный"
                            }
                        },
                        metric = metricData.getMetric(),
                        periodType = metricData.getPeriodType(),
                        notApplicable = getNotApplicableValue(
                            applicabilityStatus = metricData.getApplicabilityStatus(),
                            resumePeriod = metricData.getResumePeriod()
                        ),
                        data = agentMetrics.value.associate {
                            it.getPeriodMonth()?.let { periodMonth -> YearMonth.from(periodMonth) } to it.getMetricValue()
                        },
                        planData = agentMetrics.value.associate {
                            it.getPeriodMonth()?.let { periodMonth -> YearMonth.from(periodMonth) } to it.getTargetValue()
                        }
                    )
                }
            }
    }

    /**
     * Формирует отображаемое значение колонки "Неприменима".
     *
     * assignment отсутствует / ACTIVE -> пусто
     * PENDING -> "Согласование"
     * NOT_APPLICABLE без resumePeriod -> "Да"
     * NOT_APPLICABLE с resumePeriod -> "До <последний день месяца>"
     */
    private fun getNotApplicableValue(applicabilityStatus: String?, resumePeriod: LocalDate?): String? {
        return when (applicabilityStatus) {
            null, MetricApplicabilityStatus.ACTIVE.name -> null
            MetricApplicabilityStatus.PENDING.name -> "Согласование"

            MetricApplicabilityStatus.NOT_APPLICABLE.name -> {
                if (resumePeriod == null) {
                    "Да"
                } else {
                    "До ${YearMonth.from(resumePeriod).atEndOfMonth().format(DATE_FORMATTER)}"
                }
            }

            else -> null
        }
    }

    private fun headerColumns(periods: Set<YearMonth>): List<ExcelColumnDescription<MetricsExcelExportModel>> {
        return mutableListOf<ExcelColumnDescription<MetricsExcelExportModel>>(
            ExcelColumnDescription(
                "Блок",
                { param -> param.first.block?.let { param.third.setCellValue(it) } }
            ),
            ExcelColumnDescription(
                "Трайб",
                { param -> param.first.division?.let { param.third.setCellValue(it) } }
            ),
            ExcelColumnDescription(
                "Название агента",
                { param -> param.first.name?.let { param.third.setCellValue(it) } }
            ),
            ExcelColumnDescription(
                "CROSSGOAL",
                { param -> param.first.crossgoal?.let { param.third.setCellValue(it) } }
            ),
            ExcelColumnDescription(
                "Тип агента (автономный/copilot)",
                { param -> param.first.type?.let { param.third.setCellValue(it) } }
            ),
            ExcelColumnDescription(
                "Метрика",
                { param -> param.first.metric?.let { param.third.setCellValue(it) } }
            ),
            ExcelColumnDescription(
                "Периодичность сбора (регулярный мониторинг/вводный параметр)",
                { param -> param.first.periodType?.let { param.third.setCellValue(it) } }
            ),
            ExcelColumnDescription(
                "Неприменима",
                { param -> param.first.notApplicable?.let { param.third.setCellValue(it) } }
            )
        ).also { columns ->
            periods.forEach { period ->
                columns.add(
                    ExcelColumnDescription(
                        "факт / ${period.format(PERIOD_FORMATTER)}",
                        { param -> param.first.data[period]?.let { param.third.setCellValue(it.toDouble()) } }
                    )
                )

                columns.add(
                    ExcelColumnDescription(
                        "план / ${period.format(PERIOD_FORMATTER)}",
                        { param -> param.first.planData[period]?.let { param.third.setCellValue(it.toDouble()) } }
                    )
                )
            }
        }
    }
}

```
