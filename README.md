```java
AFTER_COMMIT listener нужен только для ускорения:

не ждать плановый scheduler
попробовать отправить сразу

Но это не единственный механизм доставки.

Реальную надежность дает вот это:

есть таблица outbox
есть периодический processor
есть retry
есть IN_PROGRESS + visibilityTimeout
```
