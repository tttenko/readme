```java
spring:
  datasource:
    url: 'jdbc:h2:mem:testdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1;DATABASE_TO_UPPER=false;INIT=CREATE ALIAS IF NOT EXISTS GEN_RANDOM_UUID FOR "java.util.UUID.randomUUID"'
    driver-class-name: org.h2.Driver
    username: sa
    password:

  liquibase:
    enabled: true

  jpa:
    hibernate:
      ddl-auto: validate
```
