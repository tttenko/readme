```java
class ConfigTest {

    @Test
    void getAsyncExecutor_shouldCreateConfiguredThreadPoolTaskExecutor() {
        AsyncConfig asyncConfig = new AsyncConfig();

        ReflectionTestUtils.setField(asyncConfig, "asyncExecutorCorePoolSize", 5);
        ReflectionTestUtils.setField(asyncConfig, "asyncExecutorMaxPoolSize", 20);
        ReflectionTestUtils.setField(asyncConfig, "asyncExecutorQueueCapacity", 200);
        ReflectionTestUtils.setField(asyncConfig, "asyncExecutorAwaitTermination", 30);

        Executor executor = asyncConfig.getAsyncExecutor();

        assertThat(executor).isInstanceOf(ThreadPoolTaskExecutor.class);

        ThreadPoolTaskExecutor taskExecutor = (ThreadPoolTaskExecutor) executor;

        assertThat(taskExecutor.getCorePoolSize()).isEqualTo(5);
        assertThat(taskExecutor.getMaxPoolSize()).isEqualTo(20);
        assertThat(taskExecutor.getThreadNamePrefix()).isEqualTo("AsyncEvent-");
        assertThat(taskExecutor.getThreadPoolExecutor()).isNotNull();
        assertThat(taskExecutor.getThreadPoolExecutor().getQueue().remainingCapacity()).isEqualTo(200);
    }

    @Test
    void schedulingConfig_shouldBeCreated() {
        SchedulingConfig schedulingConfig = new SchedulingConfig();

        assertThat(schedulingConfig).isNotNull();
        assertThat(schedulingConfig).isInstanceOf(SchedulingConfig.class);
    }
}
                
```
