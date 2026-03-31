```java
class AsyncConfigTest {

    @Test
    void getAsyncExecutor_shouldReturnConfiguredThreadPoolTaskExecutor() {
        // given
        AsyncConfig asyncConfig = new AsyncConfig();

        ReflectionTestUtils.setField(asyncConfig, "asyncExecutorCorePoolSize", 5);
        ReflectionTestUtils.setField(asyncConfig, "asyncExecutorMaxPoolSize", 20);
        ReflectionTestUtils.setField(asyncConfig, "asyncExecutorQueueCapacity", 200);
        ReflectionTestUtils.setField(asyncConfig, "asyncExecutorAwaitTermination", 30);

        // when
        Executor executor = asyncConfig.getAsyncExecutor();

        // then
        assertThat(executor).isInstanceOf(ThreadPoolTaskExecutor.class);

        ThreadPoolTaskExecutor taskExecutor = (ThreadPoolTaskExecutor) executor;

        assertThat(taskExecutor.getCorePoolSize()).isEqualTo(5);
        assertThat(taskExecutor.getMaxPoolSize()).isEqualTo(20);
        assertThat(taskExecutor.getThreadNamePrefix()).isEqualTo("AsyncEvent-");
        assertThat(taskExecutor.getThreadPoolExecutor().getQueue().remainingCapacity()).isEqualTo(200);

        ThreadPoolExecutor threadPoolExecutor = taskExecutor.getThreadPoolExecutor();
        assertThat(threadPoolExecutor).isNotNull();
    }
}

                
```
