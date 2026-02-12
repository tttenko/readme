```java
 db:
  url: jdbc:h2:mem:masterdata;MODE=PostgreSQL;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
  username: sa
  password: ""

spring:
  datasource:
    driver-class-name: org.h2.Driver

  # чтобы локально быстро поднять таблицу из Entity (а не мучать liquibase)
  jpa:
    hibernate:
      ddl-auto: create-drop
    database-platform: org.hibernate.dialect.H2Dialect

  liquibase:
    enabled: false

  h2:
    console:
      enabled: true
      path: /h2-console
```
