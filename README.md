```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Value("${spring.task.execution.pool.core-size:5}")
    private int corePoolSize;

    @Value("${spring.task.execution.pool.max-size:20}")
    private int maxPoolSize;

    @Value("${spring.task.execution.pool.queue-capacity:200}")
    private int queueCapacity;

    @Value("${spring.task.execution.thread-name-prefix:AsyncEvent-}")
    private String threadNamePrefix;

    @Value("${spring.task.execution.shutdown.await-termination:true}")
    private boolean waitForTasksToCompleteOnShutdown;

    @Value("${spring.task.execution.shutdown.await-termination-period:30s}")
    private Duration awaitTerminationPeriod;

    /**
     * Возвращает настроенный пул потоков для асинхронных задач.
     */
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(queueCapacity);
        executor.setThreadNamePrefix(threadNamePrefix);
        executor.setWaitForTasksToCompleteOnShutdown(waitForTasksToCompleteOnShutdown);
        executor.setAwaitTerminationSeconds((int) awaitTerminationPeriod.getSeconds());
        executor.initialize();
        return executor;
    }
}
```
