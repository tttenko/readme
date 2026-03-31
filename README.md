```java
spring:
  datasource:
    url: "jdbc:h2:mem:testdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1;DATABASE_TO_UPPER=false;INIT=RUNSCRIPT FROM 'classpath:schema-test-init.sql'"
    driver-class-name: org.h2.Driver
    username: sa
    password:

  liquibase:
    enabled: true

  preliquibase:
    enabled: false

  jpa:
    hibernate:
      ddl-auto: validate

app:
  kafka:
    enabled: false
```
