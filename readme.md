```java
 spring:
  datasource:
    url: jdbc:h2:file:./.h2/fxrate;MODE=PostgreSQL;AUTO_SERVER=TRUE;DATABASE_TO_UPPER=false
    driver-class-name: org.h2.Driver
    username: sa
    password:

  jpa:
    hibernate:
      ddl-auto: create-drop   # для локального smoke-теста
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
        format_sql: true
    show-sql: true

  liquibase:
    enabled: false            # чтобы не упереться в postgres-специфичный ddl

  h2:
    console:
      enabled: true
      path: /h2-console

fxrate:
  storage:
    mode: jpa    
```
