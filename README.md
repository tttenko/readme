```java
task:
    execution:
      thread-name-prefix: AsyncEvent-
      pool:
        core-size: ${app.executor.core-pool-size:5}
        max-size: ${app.executor.max-pool-size:20}
        queue-capacity: ${app.executor.queue-capacity:200}
      shutdown:
        await-termination: true
        await-termination-period: 30s

    scheduling:
      thread-name-prefix: Scheduler-
      pool:
        size: ${app.scheduler.pool-size:2}
      shutdown:
        await-termination: true
        await-termination-period: 30s
```
