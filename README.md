```java

@Component
class MetricsValuesTransferManual(
    private val metricsValuesTransferService: MetricsValuesTransferService,
    private val schedulerProperties: JiraSchedulerProperties,
    private val lockingTaskExecutor: LockingTaskExecutor,
) {

    private val log by logger()

    /**
     * Асинхронный ручной запуск переноса значений метрик
     * с распределённой блокировкой.
     *
     * Использует тот же lock, что и scheduled-запуск,
     * поэтому параллельное выполнение невозможно.
     */
    @Async
    fun run() {
        val lockAtLeastFor = schedulerProperties.lockAtLeastFor ?: Duration.ofSeconds(10)
        val lockAtMostFor = schedulerProperties.lockAtMostFor ?: Duration.ofSeconds(15)

        lockingTaskExecutor.executeWithLock(
            Runnable {
                log.info("Started manual MetricsValuesTransfer scheduler execution")

                val result = metricsValuesTransferService.transfer()

                log.info(
                    "Manual MetricsValuesTransfer scheduler completed: updated={}, skipped={}",
                    result.updatedMetricsCount,
                    result.skippedMetricsCount,
                )
            },
            LockConfiguration(
                Instant.now(),
                MetricsValuesTransferScheduler.LOCK_NAME,
                lockAtMostFor,
                lockAtLeastFor,
            )
        )
    }
}

```
